# Parte — Rede e Diagrama

## Responsável
Gabriel

## Visão geral da arquitetura

A arquitetura foi desenhada na AWS utilizando uma VPC para isolar os recursos da aplicação e separar os componentes expostos à internet dos componentes internos.

### Estrutura de rede

- **VPC:** `10.0.0.0/16`
- **Sub-rede pública A:** `10.0.1.0/24`
- **Sub-rede pública B:** `10.0.2.0/24`
- **Sub-rede privada de aplicação A:** `10.0.11.0/24`
- **Sub-rede privada de aplicação B:** `10.0.12.0/24`
- **Sub-rede privada de banco A:** `10.0.21.0/24`
- **Sub-rede privada de banco B:** `10.0.22.0/24`

A separação em duas zonas de disponibilidade melhora a disponibilidade da solução e evita concentrar todos os componentes em uma única zona.

## Componentes de rede

### Internet Gateway
Conectado à VPC para permitir que apenas os recursos das sub-redes públicas possam se comunicar diretamente com a internet.

### Sub-redes públicas
Recebem os componentes que precisam aceitar tráfego originado da internet.

Componentes:
- Application Load Balancer (ALB)
- NAT Gateway

O backend e o banco de dados não ficam nas sub-redes públicas.

### Sub-redes privadas da aplicação
Hospedam a API/backend da aplicação.

O backend:
- recebe requisições somente do Application Load Balancer;
- acessa o banco PostgreSQL;
- acessa o Amazon S3 por meio de VPC Endpoint;
- obtém credenciais pelo AWS Secrets Manager;
- envia logs e métricas para o Amazon CloudWatch.

### Sub-redes privadas de banco
Hospedam o Amazon RDS PostgreSQL.

O banco:
- não possui acesso público;
- aceita conexões PostgreSQL na porta 5432 somente do Security Group do backend;
- não aceita conexões originadas diretamente da internet nem do ALB.

## Proteção do tráfego

O fluxo de entrada é:

`Cliente -> HTTPS -> AWS WAF -> Application Load Balancer -> Backend/API`

O AWS WAF protege a aplicação contra requisições maliciosas antes que elas cheguem ao backend.

O Application Load Balancer distribui as requisições HTTPS para as instâncias ou containers responsáveis pela API.

## Armazenamento de arquivos

Os arquivos enviados pelos usuários são armazenados no Amazon S3.

O acesso do backend ao S3 utiliza um **Gateway VPC Endpoint**, evitando que esse tráfego precise sair pela internet pública.

O bucket deve permanecer privado e o acesso deve ser concedido apenas à IAM Role utilizada pelo backend.

## Acesso administrativo

A administração dos recursos internos é feita por uma **AWS Client VPN**.

Fluxo:

`Administrador -> VPN -> Recursos privados autorizados`

Não devem ser abertas portas administrativas como SSH (22) ou RDP (3389) diretamente para a internet.

## Regras de Security Groups

### SG-ALB
Entrada:
- TCP 443 a partir da internet (`0.0.0.0/0`)

Saída:
- Porta da aplicação somente para `SG-APP`

### SG-APP
Entrada:
- Porta da aplicação (ex.: TCP 8080) somente a partir de `SG-ALB`

Saída:
- TCP 5432 para `SG-RDS`
- HTTPS 443 para serviços AWS necessários

### SG-RDS
Entrada:
- TCP 5432 somente a partir de `SG-APP`

Saída:
- somente o necessário para o funcionamento do serviço

## Rotas

### Route table pública
- `10.0.0.0/16 -> local`
- `0.0.0.0/0 -> Internet Gateway`

### Route table privada da aplicação
- `10.0.0.0/16 -> local`
- `0.0.0.0/0 -> NAT Gateway`

O NAT Gateway permite que recursos privados iniciem conexões de saída quando necessário, sem permitir conexões iniciadas diretamente da internet.

### Sub-redes privadas de banco
Não necessitam de rota direta para a internet.

## Princípio de menor privilégio

A comunicação entre os componentes é liberada somente quando necessária:

- Internet não acessa diretamente backend.
- Internet não acessa diretamente banco.
- ALB não acessa diretamente banco.
- Backend é o único componente autorizado a acessar o RDS.
- Backend acessa o S3 por IAM Role e VPC Endpoint.
- Administração ocorre somente por VPN.

Essa estrutura reduz a superfície de ataque e evita exposição desnecessária dos componentes internos.
