# Rede e Diagrama

## 1. Visão geral

A aplicação é executada dentro de uma Amazon VPC para separar os componentes expostos à internet dos recursos internos. A arquitetura utiliza duas Availability Zones para evitar concentração em uma única zona.

## 2. Endereçamento

- **VPC:** `10.0.0.0/16`
- **Public Subnet A:** `10.0.1.0/24`
- **Public Subnet B:** `10.0.2.0/24`
- **Private App Subnet A:** `10.0.11.0/24`
- **Private App Subnet B:** `10.0.12.0/24`
- **Private DB Subnet A:** `10.0.21.0/24`
- **Private DB Subnet B:** `10.0.22.0/24`

## 3. Distribuição dos componentes

### Fora das sub-redes
- **AWS WAF:** associado ao Application Load Balancer. Não é colocado dentro de uma subnet.
- **Internet Gateway:** anexado à VPC. Não pertence a uma subnet.
- **Amazon S3, Secrets Manager, CloudWatch e CloudTrail:** serviços gerenciados AWS representados fora das sub-redes.

### Sub-redes públicas
- Application Load Balancer
- NAT Gateway

O ALB é o único componente da aplicação que recebe tráfego web originado da internet.

### Sub-redes privadas de aplicação
- Backend/API (EC2/ECS)

A API não possui exposição direta à internet. Recebe tráfego apenas do Security Group do ALB.

### Sub-redes privadas de banco
- Amazon RDS PostgreSQL

O banco não possui acesso público e aceita TCP 5432 apenas do Security Group da API.

## 4. Entrada da aplicação

Fluxo:

`Cliente -> HTTPS 443 -> AWS WAF -> ALB -> TCP 8080 -> Backend/API`

O AWS WAF filtra requisições maliciosas e o ALB distribui somente o tráfego aprovado para a camada de aplicação.

## 5. Rotas

### Route table pública
- `10.0.0.0/16 -> local`
- `0.0.0.0/0 -> Internet Gateway`

### Route table privada da aplicação
- `10.0.0.0/16 -> local`
- `0.0.0.0/0 -> NAT Gateway`
- rota do S3 adicionada pelo Gateway VPC Endpoint

O NAT Gateway permite conexões de saída iniciadas pelos recursos privados sem torná-los acessíveis diretamente pela internet.

### Sub-redes privadas de banco
- `10.0.0.0/16 -> local`
- sem rota padrão para Internet Gateway

## 6. Armazenamento de arquivos

Os arquivos enviados pelos usuários são armazenados em um bucket Amazon S3 privado.

A API acessa o bucket usando **Gateway VPC Endpoint para S3**, evitando que esse tráfego use a internet pública. O bucket mantém Block Public Access habilitado e restringe o acesso à role da aplicação e ao endpoint autorizado.

## 7. Acesso administrativo

O acesso administrativo utiliza **AWS Client VPN** com autenticação forte e autorização limitada.

Fluxo:

`Administrador -> Client VPN -> recurso privado autorizado`

Nenhuma porta SSH/RDP é aberta diretamente para a internet.

## 8. Security Groups

### SG-ALB
**Inbound**
- TCP 443 de `0.0.0.0/0`

**Outbound**
- TCP 8080 somente para `SG-APP`

### SG-APP
**Inbound**
- TCP 8080 somente de `SG-ALB`
- TCP 22 somente da faixa/regras autorizadas da Client VPN, se SSH for realmente necessário

**Outbound**
- TCP 5432 para `SG-RDS`
- HTTPS 443 para os serviços AWS necessários
- HTTPS 443 para S3 usando o Gateway Endpoint/prefix list

### SG-RDS
**Inbound**
- TCP 5432 somente de `SG-APP`

O RDS não recebe conexões da internet nem diretamente do ALB.

## 9. Menor privilégio

São permitidas somente as comunicações necessárias:

- Internet -> ALB: permitido em HTTPS 443
- Internet -> API: bloqueado
- Internet -> RDS: bloqueado
- ALB -> API: permitido na porta da aplicação
- ALB -> RDS: bloqueado
- API -> RDS: permitido em TCP 5432
- API -> S3: permitido via VPC Endpoint
- Administração: somente via VPN

A separação reduz a superfície de ataque e dificulta movimentação lateral.
