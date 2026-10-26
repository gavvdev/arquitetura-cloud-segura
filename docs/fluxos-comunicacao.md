# Fluxos de Comunicação

| Origem | Destino | Protocolo / Porta | Status | Finalidade |
|---|---|---|---|---|
| Cliente na internet | AWS WAF / ALB | HTTPS / TCP 443 | Permitido | Acesso seguro à aplicação |
| Internet | Backend/API | Qualquer | Bloqueado | A API não possui exposição pública |
| Internet | RDS PostgreSQL | Qualquer | Bloqueado | O banco permanece privado |
| ALB | Backend/API | TCP 8080 | Permitido | Encaminhar requisições da aplicação |
| ALB | RDS PostgreSQL | TCP 5432 | Bloqueado | O ALB não precisa acessar o banco |
| Backend/API | RDS PostgreSQL | TCP 5432 | Permitido | Leitura e gravação de dados |
| Backend/API | Amazon S3 | HTTPS / TCP 443 | Permitido | Upload/download via Gateway VPC Endpoint |
| Internet | Bucket S3 | Direto | Bloqueado | Bucket com acesso público bloqueado |
| Backend/API | AWS Secrets Manager | HTTPS / TCP 443 | Permitido | Obter segredo do banco em tempo de execução |
| Backend/API | Amazon CloudWatch | HTTPS / TCP 443 | Permitido | Envio de logs e métricas |
| Atividade/API da conta AWS | AWS CloudTrail | Serviço gerenciado | Registrado | Auditoria de ações realizadas na AWS |
| VPC | VPC Flow Logs / CloudWatch Logs | Serviço gerenciado | Registrado | Auditoria de tráfego aceito/rejeitado |
| Administrador | AWS Client VPN | TLS na porta configurada (ex.: UDP 443) | Permitido | Criar túnel administrativo seguro |
| Cliente VPN autorizado | Backend/API | TCP 22, somente se necessário | Permitido | Manutenção administrativa controlada |
| Internet | SSH/RDP de recursos privados | TCP 22/3389 | Bloqueado | Impedir administração exposta |

## Fluxo principal

1. O cliente acessa a aplicação usando HTTPS.
2. O AWS WAF inspeciona a requisição.
3. O tráfego permitido chega ao Application Load Balancer.
4. O ALB encaminha a requisição ao backend em subnet privada.
5. O backend acessa o RDS PostgreSQL somente quando necessário.
6. Arquivos são enviados ao S3 por Gateway VPC Endpoint.
7. Segredos são obtidos do Secrets Manager.
8. Logs e métricas da aplicação são enviados ao CloudWatch.
9. Eventos de API/conta AWS são registrados pelo CloudTrail.
10. VPC Flow Logs registram metadados de tráfego de rede.
11. A administração dos recursos internos ocorre somente por Client VPN.
