# pipeline-aws-sftp

<img width="3602" height="898" alt="image" src="https://github.com/user-attachments/assets/62d30469-a457-4dce-98cb-bce88cfad8b4" />

# Passo 1 — Ingestão via SFTP para S3

## Objetivo
Permitir que parceiros ou usuários externos enviem arquivos **CSV** via protocolo **SFTP**, armazenando-os diretamente no **Amazon S3** para posterior processamento.

## Como funciona

### AWS Transfer Family (SFTP)
- Fornece um endpoint **SFTP** seguro.  
- Usuários fazem login com **chave SSH** ou **credenciais configuradas**.  

### Amazon S3 (Landing Zone)
- É o destino final dos uploads.  
- Os arquivos são gravados em um **bucket S3 específico**, geralmente organizado por **pastas/prefixos**:

cliente/data/arquivo.csv

### IAM Role
- Garante que cada usuário só tenha acesso ao seu diretório no bucket S3.  
- Controla permissões de **leitura** e **escrita** no prefixo configurado.  

### Segurança e Auditoria
- Os arquivos podem ser **criptografados automaticamente** (SSE-KMS).  
- **Logs de acesso e transferência** ficam disponíveis no **CloudWatch** e no **CloudTrail**.  
