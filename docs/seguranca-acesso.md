# Parte B: Segurança de Acesso

**Banco de dados adotado:** PostgreSQL (porta 5432).

**Nomes dos componentes usados**:
WAF, ALB (Load Balancer), API (EC2/ECS em sub-rede privada), RDS (banco), S3 (arquivos), Client VPN, VPC Endpoint do S3.

---

## 1. Fluxos de comunicação

| # | Origem | Destino | Protocolo / Porta | Finalidade | Status |
|---|--------|---------|-------------------|------------|--------|
| 1 | Internet (cliente) | WAF + ALB (sub-rede pública) | HTTPS / 443 | Acesso do cliente à aplicação | Permitido |
| 2 | ALB | API (sub-rede privada) | HTTP / 8080 | Encaminhar requisições já filtradas | Permitido |
| 3 | API | RDS (sub-rede privada) | TCP / 5432 | Ler e gravar dados da aplicação | Permitido |
| 4 | API | S3 via VPC Endpoint | HTTPS / 443 | Salvar e ler arquivos sem passar pela internet | Permitido |
| 5 | API | Secrets Manager | HTTPS / 443 | Buscar a senha do banco em tempo de execução | Permitido |
| 6 | Administrador | Client VPN (endpoint) | UDP / 443 | Abrir túnel seguro, com certificado e MFA | Permitido |
| 7 | Administrador (já na VPN) | API | TCP / 22 | Manutenção do servidor | Permitido |
| 8 | Administrador (mesmo na VPN) | RDS (direto) | TCP / 5432 | Nenhuma (manutenção do banco passa pela API/backend) | **Bloqueado** |
| 9 | Internet | API (direto) | qualquer | Nenhuma | **Bloqueado** |
| 10 | Internet | RDS | qualquer | Nenhuma | **Bloqueado** |
| 11 | Internet | SSH / 22 em qualquer recurso | TCP / 22 | Nenhuma | **Bloqueado** |
| 12 | ALB | RDS | qualquer | Nenhuma (o ALB não precisa do banco) | **Bloqueado** |
| 13 | Internet | Bucket S3 (acesso público) | qualquer | Nenhuma (bucket com bloqueio de acesso público) | **Bloqueado** |
| 14 | RDS | Internet | qualquer | Nenhuma (banco não inicia conexões externas) | **Bloqueado** |

---

## 2. Security Groups

Os Security Groups são *stateful*: se a ida é permitida, a resposta volta automaticamente. Sempre que possível, a origem é outro Security Group e não um IP.

### sg-alb (Load Balancer)
- **Inbound:** HTTPS 443 de qualquer origem (0.0.0.0/0), já passando pelo WAF.
- **Outbound:** TCP 8080 somente para `sg-api`.

### sg-api (API)
- **Inbound:** TCP 8080 somente de `sg-alb`; TCP 22 somente de `sg-vpn`.
- **Outbound:** TCP 5432 para `sg-db`; HTTPS 443 para o S3 (via endpoint) e para o Secrets Manager.

### sg-db (RDS)
- **Inbound:** TCP 5432 somente de `sg-api`.
- **Outbound:** nenhuma regra. O banco não precisa iniciar conexões.

### sg-vpn (Client VPN)
- **Inbound:** UDP 443 da internet. O acesso exige certificado e MFA.
- **Outbound:** somente para `sg-api` (22).

---

## 3. IAM: identidades e permissões

### Roles para serviços

| Role | Quem assume | Permissões concedidas | Permissões que NÃO tem |
|------|-------------|-----------------------|------------------------|
| role-api | API | `s3:GetObject` e `s3:PutObject` somente em `arn:aws:s3:::bucket-uploads/*`; `secretsmanager:GetSecretValue` somente no segredo do banco; `logs:CreateLogStream` e `logs:PutLogEvents` | `s3:*`, `s3:DeleteBucket`, acesso a outros buckets e segredos, `iam:*` |
| role-flowlogs | VPC Flow Logs | Escrever logs de rede no CloudWatch Logs | Qualquer outra ação |

### Usuários e grupos

| Grupo | Quem | Permissões | Regras extras |
|-------|------|------------|---------------|
| Administradores | Equipe de infraestrutura | Gerenciar rede, instâncias e banco (sem acesso à cobrança) | MFA obrigatório; uso por role temporária |
| Desenvolvedores | Equipe de desenvolvimento | Leitura de logs (CloudWatch) e deploy no ambiente de teste | MFA obrigatório; sem acesso ao banco de produção |
| Auditoria | Segurança | Somente leitura em CloudTrail, Config e CloudWatch | MFA obrigatório; não altera nada |

**Regras gerais:** a conta root não é usada no dia a dia (apenas MFA e guardada); não se criam chaves de acesso permanentes para pessoas; política de senha forte.

---

## 4. Gestão de credenciais

- **Senha do banco:** guardada no **AWS Secrets Manager**, criptografada com KMS, com rotação automática a cada 30 dias. A API busca a senha ao iniciar usando a `role-api`; ela nunca aparece no código nem em variável de ambiente fixa.
- **Credenciais da API para a AWS:** temporárias, entregues pela role. Não existem chaves de acesso escritas no código ou no repositório.
- **Credenciais de pessoas:** login com MFA e permissões temporárias via role.
- **Pipeline (GitHub):** segredos usados em automações ficam em GitHub Secrets, nunca no repositório.

---

## 5. Acesso administrativo seguro (VPN)

- **Solução escolhida:** AWS Client VPN.
- **Problema que resolve:** o administrador precisa entrar na rede privada para manutenção, mas abrir a porta 22 para a internet exporia os servidores a varreduras e ataques de força bruta.
- **Como funciona:** o administrador abre um túnel criptografado até o endpoint da VPN, autenticando com certificado e MFA. Só depois disso ele enxerga a API (porta 22) pela rede interna. A manutenção do banco é feita a partir da API, pois o RDS aceita conexões somente dela.
- **Por que é adequada:** nenhum recurso interno recebe IP público nem porta administrativa aberta. O acesso fica ligado a identidades individuais, o que ajuda na auditoria.
- **O que fica bloqueado:** SSH direto da internet e qualquer acesso direto ao banco que não venha da API.

---

## 6. Acesso privado ao armazenamento (VPC Endpoint)

- **Tipo de endpoint:** Gateway Endpoint para S3.
- **Problema que resolve:** a API está em sub-rede privada, sem IP público. Sem o endpoint, o tráfego para o S3 precisaria sair pela internet (via NAT).
- **Como funciona:** o endpoint adiciona uma rota na tabela de rotas da sub-rede privada, e o tráfego da API para o S3 trafega pela rede interna da AWS.
- **Política do endpoint:** permite apenas `GetObject` e `PutObject` no `bucket-uploads`.
- **Camada extra:** a política do bucket nega qualquer acesso que não venha desse endpoint (`aws:SourceVpce`), o bloqueio de acesso público fica ativo e os arquivos são criptografados em repouso.
- **Por que é adequado:** os arquivos dos clientes nunca passam pela internet pública, e o acesso é limitado ao necessário.

---

## 7. Justificativas

### 7.1 Quais conexões são permitidas e quais são bloqueadas, e por quê?

Só passam as conexões necessárias para a aplicação funcionar:

- Cliente → ALB (443)
- ALB → API (8080)
- API → banco (5432)
- API → S3 e Secrets Manager (443)
- Administrador → API, somente pela VPN

Todo o resto é bloqueado por padrão, em especial qualquer acesso direto da internet à API, ao banco e à porta 22. O banco aceita conexões apenas da API.

**Por quê:** mesmo que um componente seja comprometido, o atacante tem poucos caminhos para andar pela rede.

### 7.2 Como o acesso administrativo é protegido?

- Acesso por Client VPN, com certificado e MFA.
- Nenhuma porta SSH aberta na internet.
- Cada administrador tem identidade própria.
- O acesso aos recursos internos só existe depois que o túnel está aberto.

**Resultado:** a superfície de ataque diminui e fica registrado quem entrou e quando.

### 7.3 Como as permissões dos serviços e usuários são limitadas?

Seguindo o princípio do menor privilégio:

- **Serviços:** a API usa uma role que permite apenas leitura e escrita no bucket dela e a leitura de um segredo específico.
- **Pessoas:** ficam em grupos com permissões mínimas, com MFA obrigatório.
- **Credenciais:** não há chaves permanentes nem uso da conta root.

### 7.4 Como esta parte reduz o risco de exposição de dados e acesso não autorizado?

- O banco e a API não têm IP público.
- O S3 não é acessível pela internet.
- As senhas ficam no Secrets Manager, com rotação.
- As comunicações são limitadas por Security Groups encadeados.

**Resultado:** cada camada exige passar pela anterior, e uma falha isolada não dá acesso direto aos dados.

---

## 8. Pontos a alinhar com o grupo

- **Entrada do Client VPN:** definir como representar o acesso à VPN no diagrama e nos Security Groups. O endpoint recebe conexões na porta 443 (certificado e MFA), e o Security Group controla o tráfego dos clientes VPN para dentro da VPC.
- **API → Secrets Manager:** definir se a API acessa o serviço por NAT Gateway ou por VPC Endpoint de interface. Depende do diagrama (Parte A).