# Segurança de Acesso

## 1. Banco de dados

**Serviço:** Amazon RDS  
**Engine:** PostgreSQL  
**Porta:** TCP 5432  
**Acesso público:** desabilitado

O banco é implantado em sub-redes privadas e aceita conexões somente do Security Group da camada de aplicação.

## 2. IAM e menor privilégio

### Role da API

A aplicação utiliza uma IAM Role, sem credenciais permanentes no código.

Permissões mínimas esperadas:
- `s3:GetObject` e `s3:PutObject` somente no bucket de uploads;
- `secretsmanager:GetSecretValue` somente no segredo necessário;
- permissões mínimas de CloudWatch Logs para geração de logs.

A role não deve possuir `s3:*`, `iam:*` ou acesso genérico a todos os segredos.

### Usuários administrativos

- MFA obrigatório;
- conta root não utilizada no dia a dia;
- privilégios administrativos concedidos somente quando necessários;
- desenvolvedores não recebem acesso direto ao banco de produção;
- auditoria recebe somente leitura nos serviços de logs.

## 3. Gestão de credenciais

- A senha do PostgreSQL fica no **AWS Secrets Manager**.
- A aplicação recupera a credencial usando sua IAM Role.
- Não existem chaves AWS ou senhas no GitHub.
- Segredos de CI/CD devem utilizar GitHub Secrets ou mecanismo equivalente.
- Rotação de segredos deve ser habilitada quando aplicável.

## 4. Client VPN

O AWS Client VPN fornece acesso administrativo à rede privada.

### Proteções
- autenticação forte (certificados e/ou integração de identidade);
- MFA quando suportado pelo método de autenticação adotado;
- regras de autorização limitadas às redes necessárias;
- Security Groups restringindo o tráfego dos clientes VPN;
- nenhuma porta administrativa aberta diretamente à internet.

O endpoint da Client VPN escuta na porta/protocolo configurados (por exemplo, UDP 443). O Security Group associado é usado para limitar o tráfego entre os clientes VPN e os recursos da VPC; ele não deve ser descrito como se fosse um Security Group de uma instância pública comum.

## 5. S3 privado

- Block Public Access habilitado;
- criptografia em repouso;
- acesso da API por Gateway VPC Endpoint;
- política de bucket restringindo ações ao necessário;
- política do endpoint limitada ao bucket da aplicação.

## 6. Secrets Manager

A API pode acessar o Secrets Manager:
- via saída HTTPS controlada pelo NAT Gateway; ou
- preferencialmente, caso o grupo queira eliminar esse caminho público, por **Interface VPC Endpoint para Secrets Manager**.

Para manter o diagrama simples, esta entrega considera HTTPS por saída controlada/NAT, enquanto o S3 usa explicitamente Gateway VPC Endpoint.

## 7. Conexões bloqueadas

- Internet -> API
- Internet -> RDS
- Internet -> S3 público
- Internet -> SSH/RDP
- ALB -> RDS
- qualquer acesso ao banco que não venha da aplicação

Esses bloqueios implementam defesa em profundidade e reduzem a movimentação lateral.
