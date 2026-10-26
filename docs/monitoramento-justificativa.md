# Monitoramento e Justificativa Técnica

Este documento cobre o monitoramento/auditoria e a justificativa técnica das decisões da arquitetura. Ele segue o que está no diagrama (`diagrama/arquitetura-cloud-segura.drawio`) e nos demais arquivos de `docs/`.

---

## 1. Monitoramento e auditoria

A arquitetura usa três serviços que cobrem camadas diferentes. Cada um responde a uma pergunta distinta:

| Serviço | Camada | Pergunta que responde |
|---|---|---|
| **Amazon CloudWatch** | Aplicação e infraestrutura | "O sistema está saudável? O que a aplicação registrou?" |
| **AWS CloudTrail** | Ações na conta AWS | "Quem fez qual alteração, quando e de onde?" |
| **VPC Flow Logs** | Rede | "Quais conexões foram aceitas ou rejeitadas?" |

### 1.1 Amazon CloudWatch

Recebe logs e métricas da API (enviados por HTTPS 443, pela saída controlada do NAT Gateway) e coleta métricas do ALB e do RDS. Também recebe os Flow Logs e é o local onde ficam os alarmes.

Alarmes sugeridos:

| Alarme | Fonte | Indica |
|---|---|---|
| Taxa elevada de erros HTTP 5xx | Métricas do ALB | Falha na aplicação ou ataque |
| Hosts da API sem saúde (unhealthy) | Métricas do ALB | Backend indisponível |
| CPU ou número de conexões do RDS acima do normal | Métricas do RDS | Sobrecarga ou uso anormal do banco |
| Volume anormal de requisições | Métricas do ALB/WAF | Possível abuso ou tentativa de ataque |
| Muitos registros REJECT nas portas 5432 ou 22 | Flow Logs (filtro de métrica) | Tentativa de acesso ao banco ou à administração |
| Alteração de Security Group, rota ou política IAM | CloudTrail (filtro de métrica) | Mudança de configuração que merece revisão |
| Login do usuário root | CloudTrail (filtro de métrica) | Uso indevido da conta mais privilegiada |

### 1.2 AWS CloudTrail

Registra as chamadas de API e as ações administrativas na conta AWS. Ajuda a responder:

- quem alterou um Security Group;
- quem modificou uma política IAM;
- quem alterou um bucket ou uma configuração do RDS;
- quem leu o segredo do banco no Secrets Manager (`GetSecretValue`);
- quando cada alteração foi realizada e de qual IP.

O CloudTrail é o mecanismo de auditoria de **ações na AWS**, e não o destino dos logs da aplicação (esses vão para o CloudWatch).

### 1.3 VPC Flow Logs

Registram metadados de tráfego (origem, destino, porta, protocolo, aceito ou rejeitado), gravados no CloudWatch Logs pela `role-flowlogs`, que só pode escrever logs.

Ajudam a identificar:

- tentativas de conexão bloqueadas (por exemplo, acesso direto à porta 5432 ou 22);
- comunicação inesperada entre componentes;
- origem e destino de tráfego suspeito.

Os Flow Logs **não** mostram o conteúdo dos pacotes, apenas os metadados da conexão.

### 1.4 Fluxos de comunicação do monitoramento

| Origem | Destino | Protocolo / Porta | Status | Finalidade |
|---|---|---|---|---|
| Backend/API | NAT Gateway → Amazon CloudWatch | HTTPS / TCP 443 | Permitido | Enviar logs e métricas da aplicação |
| Backend/API | NAT Gateway → AWS Secrets Manager | HTTPS / TCP 443 | Permitido | Obter o segredo do banco (a leitura fica registrada no CloudTrail) |
| ALB e RDS | Amazon CloudWatch | Serviço gerenciado | Registrado | Métricas de disponibilidade e desempenho |
| Atividade/API da conta AWS | AWS CloudTrail | Serviço gerenciado | Registrado | Auditoria de ações realizadas na AWS |
| VPC | VPC Flow Logs → CloudWatch Logs | Serviço gerenciado | Registrado | Auditoria de tráfego aceito e rejeitado |
| CloudWatch Alarms | Equipe responsável | Serviço gerenciado | Permitido | Notificar eventos relevantes |
| Internet | CloudWatch / CloudTrail / Flow Logs (escrita direta) | Qualquer | Bloqueado | Só os serviços e roles autorizados gravam logs |

---

## 2. Justificativa dos componentes

### VPC
Cria isolamento lógico para os recursos da aplicação e delimita o ambiente (`10.0.0.0/16`).

### Sub-redes públicas
São usadas apenas para componentes que precisam de conectividade pública: o ALB e o NAT Gateway.

### Sub-redes privadas de aplicação
Protegem o backend de conexões iniciadas diretamente pela internet. A API não tem IP público.

### Sub-redes privadas de banco
Mantêm o RDS isolado, sem rota para a internet, e acessível somente pela aplicação.

### AWS WAF
Adiciona uma camada de proteção para requisições web (como SQL injection e XSS) antes do processamento pelo backend. Fica associado ao ALB.

### Application Load Balancer
É **um único** load balancer associado às sub-redes públicas das duas Availability Zones. Recebe HTTPS e distribui as requisições para os backends privados disponíveis nas duas zonas, o que também dá resiliência.

### NAT Gateway
Permite saída controlada dos recursos privados para serviços que não usam endpoint privado (neste projeto, Secrets Manager e CloudWatch), sem oferecer entrada pública para esses recursos. Fica em sub-rede pública.

### Gateway VPC Endpoint para S3
Resolve o problema de acessar o S3 a partir de uma sub-rede privada sem passar pela internet pública nem consumir o NAT. A política do endpoint limita o acesso ao bucket da aplicação.

### Amazon S3 privado
Serviço adequado para os arquivos enviados pelos usuários, com Block Public Access, criptografia em repouso e acesso restrito à role da API.

### Client VPN
Resolve o acesso administrativo seguro: o administrador entra na rede privada por túnel criptografado e autenticado, sem expor SSH/RDP à internet.

### IAM Roles e Secrets Manager
Eliminam credenciais AWS permanentes no código e restringem o acesso aos recursos necessários. A senha do banco fica no Secrets Manager, e a API a obtém em tempo de execução por meio de uma role.

### CloudWatch, CloudTrail e VPC Flow Logs
Fornecem a visibilidade necessária para operar o sistema e auditar incidentes (seção 1).

---

## 3. Conexões permitidas e bloqueadas

A arquitetura segue **negação por padrão**: tudo o que não foi explicitamente liberado em um Security Group é bloqueado.

**Permitidas:**
- cliente → ALB em HTTPS (443);
- ALB → API somente na porta da aplicação (8080);
- API → RDS somente em 5432;
- API → S3 em HTTPS, pelo Gateway VPC Endpoint;
- API → Secrets Manager e CloudWatch em HTTPS, pela saída do NAT Gateway;
- administrador → API (TCP 22, se necessário) somente após conectar na Client VPN.

**Bloqueadas:**
- internet → API;
- internet → RDS;
- ALB → RDS;
- RDS → internet;
- acesso público ao S3;
- SSH/RDP direto da internet;
- administrador → RDS direto (a manutenção do banco passa pela API).

Os Security Groups referenciam outros Security Groups (por exemplo, o RDS aceita 5432 somente de `SG-APP`) em vez de faixas de IP amplas. Isso mantém a regra correta mesmo quando os IPs mudam.

---

## 4. Como o acesso administrativo é protegido

- Nenhuma porta SSH/RDP está aberta na internet.
- O administrador conecta na **Client VPN** com autenticação forte (certificado e/ou identidade, com MFA quando suportado).
- Só depois do túnel aberto ele alcança a API (TCP 22), e o `SG-APP` aceita a porta 22 apenas do `SG-VPN`.
- Cada administrador tem identidade própria, o que permite saber quem acessou e quando.
- As ações feitas na conta AWS ficam registradas no CloudTrail.

---

## 5. Como as permissões de serviços e usuários são limitadas

- **API:** usa a `role-api`, que permite apenas `GetObject`/`PutObject` no bucket de uploads, `GetSecretValue` no segredo do banco e envio de logs. Não tem `iam:*` nem acesso a outros buckets ou segredos.
- **Flow Logs:** a `role-flowlogs` só escreve logs de rede no CloudWatch Logs.
- **Pessoas:** ficam em grupos (Administradores, Desenvolvedores, Auditoria) com MFA obrigatório. O grupo de Auditoria tem somente leitura em CloudTrail e CloudWatch.
- **Credenciais:** não há chaves permanentes no código, e a conta root não é usada no dia a dia.

Os detalhes completos estão em `docs/seguranca-acesso.md`.

---

## 6. Como a arquitetura reduz riscos de exposição de dados e acesso não autorizado

| Risco | Como é reduzido |
|---|---|
| Ataques à aplicação web | WAF na frente do ALB |
| Acesso direto ao banco | RDS em sub-rede privada, sem acesso público, aceitando só `SG-APP` |
| Acesso direto à API | Sem IP público; só aceita tráfego do ALB e da VPN |
| Exposição dos arquivos | S3 privado, com política que exige o endpoint e acesso só da role da API |
| Vazamento de credenciais | Secrets Manager com rotação e IAM Roles (sem chaves no código) |
| Acesso administrativo indevido | Client VPN com autenticação forte e SG restritivo |
| Movimentação lateral | Security Groups encadeados e permissões mínimas |
| Ações sem rastro | CloudTrail, Flow Logs e CloudWatch permitem investigar o que aconteceu |

---

## 7. Como os registros ajudam a identificar atividades suspeitas

Os três serviços se complementam:

- **CloudWatch:** mostra a saúde da aplicação e da infraestrutura (picos de erro, queda do backend, uso anormal do banco).
- **CloudTrail:** mostra quem realizou ações administrativas ou de API na AWS (por exemplo, uma regra de Security Group aberta ao mundo, uma política IAM alterada ou uma leitura incomum do segredo do banco).
- **VPC Flow Logs:** mostra quais fluxos de rede foram aceitos ou rejeitados (por exemplo, várias rejeições na porta 5432 ou 22 indicam tentativa de acesso indevido).

**Exemplo de correlação:** os Flow Logs mostram tentativas rejeitadas no banco e o CloudTrail mostra, no mesmo horário, a alteração de um Security Group. Juntos, indicam que alguém pode ter tentado abrir um caminho para o RDS. Com esses registros é possível reconstruir a linha do tempo do incidente.