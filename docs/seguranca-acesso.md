# Segurança de Acesso

Esta parte define quem pode acessar o quê na arquitetura: fluxos permitidos e bloqueados, Security Groups, identidades e permissões (IAM), credenciais, acesso administrativo e acesso privado ao armazenamento.

**Componentes citados:** AWS WAF, Application Load Balancer (ALB), Backend/API (EC2 ou ECS), Amazon RDS PostgreSQL, Amazon S3, Gateway VPC Endpoint para S3, NAT Gateway, AWS Secrets Manager, AWS Client VPN.

**Banco de dados:** Amazon RDS PostgreSQL (TCP 5432), sem acesso público, em sub-redes privadas de banco. Aceita conexões somente do Security Group da aplicação (`SG-APP`).

---

## 1. Fluxos de comunicação

| # | Origem | Destino | Protocolo / Porta | Finalidade | Status |
|---|--------|---------|-------------------|------------|--------|
| 1 | Internet (cliente) | AWS WAF + ALB (sub-rede pública) | HTTPS / 443 | Acesso do cliente à aplicação | Permitido |
| 2 | ALB | Backend/API (sub-rede privada) | HTTP / 8080 | Encaminhar requisições já filtradas | Permitido |
| 3 | Backend/API | RDS PostgreSQL (sub-rede privada de banco) | TCP / 5432 | Ler e gravar dados da aplicação | Permitido |
| 4 | Backend/API | S3 via Gateway VPC Endpoint | HTTPS / 443 | Salvar e ler arquivos sem passar pela internet | Permitido |
| 5 | Backend/API | Secrets Manager (saída pelo NAT Gateway) | HTTPS / 443 | Buscar a senha do banco em tempo de execução | Permitido |
| 6 | Backend/API | CloudWatch | HTTPS / 443 | Enviar logs e métricas da aplicação | Permitido |
| 7 | Administrador | Client VPN (endpoint) | UDP / 443 (porta configurável) | Abrir túnel administrativo seguro | Permitido |
| 8 | Administrador (já na VPN) | Backend/API | TCP / 22, somente se necessário | Manutenção do servidor | Permitido |
| 9 | Administrador (mesmo na VPN) | RDS (direto) | TCP / 5432 | Nenhuma (manutenção do banco passa pela API/backend) | **Bloqueado** |
| 10 | Internet | Backend/API (direto) | qualquer | Nenhuma | **Bloqueado** |
| 11 | Internet | RDS | qualquer | Nenhuma | **Bloqueado** |
| 12 | Internet | SSH / RDP de recursos privados | TCP / 22 e 3389 | Nenhuma | **Bloqueado** |
| 13 | ALB | RDS | qualquer | Nenhuma (o ALB não precisa do banco) | **Bloqueado** |
| 14 | Internet | Bucket S3 (acesso público) | qualquer | Nenhuma (Block Public Access ativo) | **Bloqueado** |
| 15 | RDS | Internet | qualquer | Nenhuma (o banco não inicia conexões externas) | **Bloqueado** |

---

## 2. Security Groups

Os Security Groups são *stateful*: se a ida é permitida, a resposta volta automaticamente. Sempre que possível, a origem é outro Security Group e não um IP.

### SG-ALB (Load Balancer)
- **Inbound:** HTTPS 443 de qualquer origem (0.0.0.0/0), já passando pelo WAF.
- **Outbound:** TCP 8080 somente para `SG-APP`.

### SG-APP (Backend/API)
- **Inbound:** TCP 8080 somente de `SG-ALB`; TCP 22 somente de `SG-VPN`, se SSH for realmente necessário.
- **Outbound:** TCP 5432 para `SG-RDS`; HTTPS 443 para o S3 (prefix list do Gateway Endpoint) e para os serviços AWS necessários (Secrets Manager e CloudWatch, pela saída controlada do NAT Gateway).

### SG-RDS (Banco de dados)
- **Inbound:** TCP 5432 somente de `SG-APP`.
- **Outbound:** nenhuma regra. O banco não precisa iniciar conexões.

### SG-VPN (Client VPN)
O Security Group associado ao Client VPN não funciona como o de uma instância pública. O endpoint escuta na porta e protocolo configurados (por exemplo, UDP 443), e a proteção dessa entrada vem da autenticação forte. O Security Group serve para limitar o tráfego **dos clientes VPN para dentro da VPC**:
- **Outbound (clientes VPN → VPC):** somente TCP 22 para `SG-APP`, quando necessário.

---

## 3. IAM: identidades e permissões

### Roles para serviços

| Role | Quem assume | Permissões concedidas | Permissões que NÃO tem |
|------|-------------|-----------------------|------------------------|
| role-api | Backend/API | `s3:GetObject` e `s3:PutObject` somente em `arn:aws:s3:::bucket-uploads/*`; `secretsmanager:GetSecretValue` somente no segredo do banco; `logs:CreateLogStream` e `logs:PutLogEvents` | `s3:*`, `s3:DeleteBucket`, `iam:*`, acesso genérico a todos os segredos, acesso a outros buckets |
| role-flowlogs | VPC Flow Logs | Escrever logs de rede no CloudWatch Logs | Qualquer outra ação |

### Usuários e grupos

| Grupo | Quem | Permissões | Regras extras |
|-------|------|------------|---------------|
| Administradores | Equipe de infraestrutura | Gerenciar rede, instâncias e banco (sem acesso à cobrança), concedidas somente quando necessárias | MFA obrigatório; uso por role temporária |
| Desenvolvedores | Equipe de desenvolvimento | Leitura de logs (CloudWatch) e deploy no ambiente de teste | MFA obrigatório; sem acesso direto ao banco de produção |
| Auditoria | Segurança | Somente leitura em CloudTrail, Config e CloudWatch | MFA obrigatório; não altera nada |

**Regras gerais:** a conta root não é usada no dia a dia (apenas MFA e guardada); não se criam chaves de acesso permanentes para pessoas; política de senha forte.

---

## 4. Gestão de credenciais

- **Senha do banco:** guardada no **AWS Secrets Manager**, criptografada com KMS, com rotação habilitada (por exemplo, a cada 30 dias). A API busca a senha usando a `role-api`; ela nunca aparece no código nem em variável de ambiente fixa.
- **Credenciais da API para a AWS:** temporárias, entregues pela role. Não existem chaves de acesso escritas no código ou no repositório.
- **Credenciais de pessoas:** login com MFA e permissões temporárias via role.
- **Pipeline (GitHub):** segredos usados em automações ficam em GitHub Secrets ou mecanismo equivalente, nunca no repositório. Não existem chaves AWS ou senhas no GitHub.

---

## 5. Acesso administrativo seguro (Client VPN)

- **Solução escolhida:** AWS Client VPN.
- **Problema que resolve:** o administrador precisa entrar na rede privada para manutenção, mas abrir a porta 22 para a internet exporia os servidores a varreduras e ataques de força bruta.
- **Como funciona:** o administrador abre um túnel criptografado até o endpoint da VPN. Só depois disso ele enxerga a API (porta 22, se necessário) pela rede interna. A manutenção do banco é feita a partir da API, pois o RDS aceita conexões somente dela.
- **Proteções:**
  - autenticação forte, com certificados e/ou integração de identidade;
  - MFA quando suportado pelo método de autenticação adotado;
  - regras de autorização limitadas às redes necessárias;
  - Security Group restringindo o tráfego dos clientes VPN;
  - nenhuma porta administrativa aberta diretamente à internet.
- **Por que é adequada:** nenhum recurso interno recebe IP público nem porta administrativa aberta. O acesso fica ligado a identidades individuais, o que ajuda na auditoria.
- **O que fica bloqueado:** SSH/RDP direto da internet e qualquer acesso direto ao banco que não venha da API.

---

## 6. Acesso privado ao armazenamento (S3 e VPC Endpoint)

- **Tipo de endpoint:** Gateway VPC Endpoint para S3.
- **Problema que resolve:** a API está em sub-rede privada, sem IP público. Sem o endpoint, o tráfego para o S3 precisaria sair pela internet (via NAT).
- **Como funciona:** o endpoint adiciona uma rota na tabela de rotas da sub-rede privada, e o tráfego da API para o S3 trafega pela rede interna da AWS.
- **Proteções do bucket:**
  - Block Public Access habilitado;
  - criptografia em repouso;
  - política do bucket limitando as ações ao necessário e negando acesso que não venha do endpoint (`aws:SourceVpce`);
  - política do endpoint limitada ao bucket da aplicação (`GetObject` e `PutObject` no `bucket-uploads`).
- **Por que é adequado:** os arquivos dos clientes nunca passam pela internet pública, e o acesso é limitado ao necessário.

---

## 7. Acesso da API ao Secrets Manager

- **Solução adotada:** HTTPS por saída controlada, pelo NAT Gateway. O NAT permite conexões iniciadas pela API sem tornar a API acessível a partir da internet.
- **Alternativa:** um Interface VPC Endpoint para o Secrets Manager eliminaria esse caminho de saída. Para manter o diagrama simples, esta entrega usa o NAT, enquanto o S3 usa explicitamente o Gateway Endpoint.

---

## 8. Justificativas

### 8.1 Quais conexões são permitidas e quais são bloqueadas, e por quê?

Só passam as conexões necessárias para a aplicação funcionar:

- Cliente → ALB (443)
- ALB → API (8080)
- API → banco (5432)
- API → S3 (443, via Gateway Endpoint)
- API → Secrets Manager e CloudWatch (443)
- Administrador → API, somente pela VPN

Todo o resto é bloqueado por padrão, em especial qualquer acesso direto da internet à API, ao banco e às portas de administração. O banco aceita conexões apenas da API.

**Por quê:** mesmo que um componente seja comprometido, o atacante tem poucos caminhos para andar pela rede.

### 8.2 Como o acesso administrativo é protegido?

- Acesso por Client VPN, com autenticação forte (certificado e/ou identidade) e MFA quando suportado.
- Nenhuma porta SSH/RDP aberta na internet.
- Cada administrador tem identidade própria.
- O acesso aos recursos internos só existe depois que o túnel está aberto, e as regras de autorização limitam as redes alcançadas.

**Resultado:** a superfície de ataque diminui e fica registrado quem entrou e quando.

### 8.3 Como as permissões dos serviços e usuários são limitadas?

Seguindo o princípio do menor privilégio:

- **Serviços:** a API usa uma role que permite apenas leitura e escrita no bucket dela, a leitura de um segredo específico e o envio de logs.
- **Pessoas:** ficam em grupos com permissões mínimas, com MFA obrigatório e sem acesso direto ao banco de produção.
- **Credenciais:** não há chaves permanentes nem uso da conta root.

### 8.4 Como esta parte reduz o risco de exposição de dados e acesso não autorizado?

- O banco e a API não têm IP público.
- O S3 não é acessível pela internet.
- As senhas ficam no Secrets Manager, com rotação.
- As comunicações são limitadas por Security Groups encadeados.

**Resultado:** cada camada exige passar pela anterior, e uma falha isolada não dá acesso direto aos dados.
