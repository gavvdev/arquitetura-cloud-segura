# Arquitetura Cloud Segura — Parte de Rede e Diagrama

Arquivos preparados para a parte de **rede e diagrama** da atividade.

## Arquivos

- `arquitetura-cloud-segura.drawio` — diagrama editável no Draw.io/diagrams.net.
- `rede-e-diagrama.md` — descrição técnica da rede e decisões de segurança.
- `fluxos-comunicacao.md` — tabela com conexões permitidas e bloqueadas.

## Arquitetura adotada

Fluxo principal:

`Internet -> AWS WAF -> Application Load Balancer -> Backend/API -> RDS PostgreSQL`

Também são utilizados:

- VPC
- Sub-redes públicas e privadas
- Internet Gateway
- NAT Gateway
- AWS Client VPN
- Amazon S3
- Gateway VPC Endpoint para S3
- AWS Secrets Manager
- Amazon CloudWatch
- AWS CloudTrail
- IAM Roles

## Como abrir o diagrama

1. Acesse https://app.diagrams.net/
2. Escolha **Abrir diagrama existente**.
3. Selecione `arquitetura-cloud-segura.drawio`.
4. O arquivo também pode ser aberto no VS Code com uma extensão compatível com Draw.io.

## Sugestão de commit

```bash
git add .
git commit -m "feat: adiciona arquitetura de rede e diagrama cloud"
git push
```
