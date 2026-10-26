# Monitoramento e Justificativa Técnica

## 1. Monitoramento e auditoria

### Amazon CloudWatch
Utilizado para:
- logs da aplicação;
- métricas do backend;
- métricas do ALB e do RDS;
- alarmes de disponibilidade, erros e uso de recursos.

Exemplos de alertas:
- taxa elevada de erros HTTP 5xx;
- backend indisponível;
- aumento anormal de CPU/conexões no RDS;
- comportamento anormal de requisições.

### AWS CloudTrail
Registra atividades de API e ações administrativas realizadas na conta AWS.

Ajuda a responder perguntas como:
- quem alterou um Security Group;
- quem modificou uma política IAM;
- quem alterou um bucket;
- quando determinada alteração foi realizada.

CloudTrail é tratado como mecanismo de auditoria de ações AWS, e não como destino comum dos logs da aplicação.

### VPC Flow Logs
Registra metadados de tráfego de rede aceito e rejeitado.

Ajuda a identificar:
- tentativas de conexão bloqueadas;
- comunicação inesperada entre componentes;
- origem e destino de tráfego suspeito.

## 2. Justificativa dos componentes

### VPC
Cria isolamento lógico para os recursos da aplicação.

### Sub-redes públicas
São usadas apenas para componentes que precisam de conectividade pública, como o ALB e o NAT Gateway.

### Sub-redes privadas de aplicação
Protegem o backend de conexões iniciadas diretamente pela internet.

### Sub-redes privadas de banco
Mantêm o RDS isolado e acessível somente pela aplicação.

### AWS WAF
Adiciona uma camada de proteção para requisições web antes do processamento pelo backend.

### Application Load Balancer
Recebe HTTPS e distribui as requisições para a camada privada de aplicação.

### NAT Gateway
Permite saída controlada dos recursos privados quando um serviço externo precisa ser acessado, sem oferecer entrada pública para esses recursos.

### Gateway VPC Endpoint para S3
Permite acesso privado ao S3 sem depender do NAT Gateway ou da internet pública.

### Client VPN
Permite administração dos recursos privados sem expor SSH/RDP à internet.

### IAM Roles e Secrets Manager
Eliminam credenciais AWS permanentes no código e restringem o acesso aos recursos necessários.

## 3. Conexões permitidas e bloqueadas

A arquitetura segue negação por padrão.

Permitidas:
- cliente -> ALB em HTTPS;
- ALB -> API somente na porta da aplicação;
- API -> RDS somente em 5432;
- API -> S3 e serviços autorizados em HTTPS;
- administrador -> recursos autorizados somente após VPN.

Bloqueadas:
- internet -> API;
- internet -> RDS;
- ALB -> RDS;
- acesso público ao S3;
- SSH/RDP direto da internet.

## 4. Redução de risco

A arquitetura reduz exposição porque:
- API e banco não têm acesso público;
- o banco aceita somente a origem esperada;
- o S3 permanece privado;
- segredos não ficam no código;
- o WAF filtra tráfego web;
- a administração exige canal privado;
- logs de aplicação, rede e auditoria permitem investigar ações suspeitas.

## 5. Como os registros ajudam na segurança

CloudWatch, CloudTrail e VPC Flow Logs cobrem camadas diferentes:

- **CloudWatch:** saúde da aplicação e infraestrutura;
- **CloudTrail:** quem realizou ações administrativas/API na AWS;
- **VPC Flow Logs:** quais fluxos de rede foram aceitos ou rejeitados.

A correlação desses registros ajuda a detectar comportamento anormal e reconstruir incidentes.
