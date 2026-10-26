# Fluxos de Comunicação

| Origem | Destino | Protocolo/Porta | Permitido | Finalidade |
|---|---|---:|:---:|---|
| Cliente Internet | AWS WAF / ALB | HTTPS TCP 443 | Sim | Acesso seguro à aplicação |
| Internet | Backend/API | Qualquer | Não | Backend não deve ficar publicamente acessível |
| Internet | RDS PostgreSQL | TCP 5432 | Não | Banco deve permanecer privado |
| AWS WAF / ALB | Backend/API | TCP 8080 | Sim | Encaminhamento das requisições da aplicação |
| ALB | RDS PostgreSQL | TCP 5432 | Não | Banco só pode ser acessado pelo backend |
| Backend/API | RDS PostgreSQL | TCP 5432 | Sim | Consultas e persistência de dados |
| Backend/API | Amazon S3 | HTTPS TCP 443 | Sim | Upload/download de arquivos via VPC Endpoint |
| Cliente Internet | Amazon S3 privado | Direto | Não | Bucket não deve ficar público |
| Administrador | AWS Client VPN | VPN/TLS | Sim | Entrada administrativa segura |
| VPN | Recursos privados autorizados | Conforme necessidade | Sim | Manutenção administrativa |
| Internet | SSH/RDP dos recursos privados | TCP 22/3389 | Não | Evitar administração exposta |
| Backend/API | Secrets Manager | HTTPS TCP 443 | Sim | Recuperação segura de credenciais |
| Componentes AWS | CloudWatch/CloudTrail | HTTPS TCP 443 | Sim | Logs, métricas e auditoria |

## Fluxo principal

1. O cliente acessa a aplicação utilizando HTTPS.
2. O tráfego passa pelo AWS WAF.
3. O WAF encaminha o tráfego permitido ao Application Load Balancer.
4. O ALB envia a requisição ao backend localizado em sub-rede privada.
5. O backend consulta o Amazon RDS PostgreSQL quando necessário.
6. Para arquivos, o backend acessa o Amazon S3 por VPC Endpoint.
7. Logs e eventos são enviados para CloudWatch e CloudTrail.
8. Administradores acessam recursos privados somente através da VPN.
