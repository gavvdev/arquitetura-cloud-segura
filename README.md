# Arquitetura Cloud Segura

Projeto acadêmico de uma arquitetura web segura na AWS, com foco em isolamento de rede, menor privilégio, proteção das comunicações, armazenamento privado, acesso administrativo seguro e monitoramento.

## Arquitetura adotada

Fluxo principal:

`Cliente -> HTTPS 443 -> AWS WAF -> Application Load Balancer -> Backend/API -> Amazon RDS PostgreSQL`

Fluxos complementares:

- Backend/API -> Amazon S3 por Gateway VPC Endpoint
- Backend/API -> AWS Secrets Manager por HTTPS
- Backend/API -> Amazon CloudWatch para logs e métricas
- Atividade da conta/API AWS -> AWS CloudTrail para auditoria
- VPC -> VPC Flow Logs -> CloudWatch Logs
- Administrador -> AWS Client VPN -> recursos privados autorizados

## Estrutura

```text
arquitetura-cloud-segura/
├── diagrama/
│   └── arquitetura-cloud-segura.drawio
├── docs/
│   ├── fluxos-comunicacao.md
│   ├── monitoramento-justificativa.md
│   ├── rede-e-diagrama.md
│   └── seguranca-acesso.md
└── README.md
```

## Componentes principais

- Amazon VPC
- 2 Availability Zones
- Sub-redes públicas
- Sub-redes privadas de aplicação
- Sub-redes privadas de banco
- Internet Gateway
- NAT Gateway
- AWS WAF
- Application Load Balancer
- Backend/API em EC2 ou ECS
- Amazon RDS PostgreSQL
- Amazon S3 privado
- Gateway VPC Endpoint para S3
- AWS Client VPN
- AWS Secrets Manager
- IAM Roles
- Amazon CloudWatch
- AWS CloudTrail
- VPC Flow Logs

## Banco de dados

Foi adotado **Amazon RDS PostgreSQL**, sem acesso público. O Security Group do banco permite TCP 5432 somente a partir do Security Group da aplicação.

## Como abrir o diagrama

Abra `diagrama/arquitetura-cloud-segura.drawio` no Draw.io/diagrams.net ou em uma extensão Draw.io compatível com VS Code.

## Entrega

O repositório contém:
1. diagrama editável;
2. justificativas técnicas;
3. fluxos permitidos e bloqueados;
4. estratégia de identidade, credenciais e menor privilégio;
5. estratégia de monitoramento e auditoria.
