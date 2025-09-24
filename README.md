# Pipeline AWS ingestão de dados com SFTP

<img width="3602" height="898" alt="image" src="https://github.com/user-attachments/assets/62d30469-a457-4dce-98cb-bce88cfad8b4" />

Este projeto consiste na implementação de um pipeline de dados na AWS para ingestão automática de arquivos CSV enviados via SFTP, com posterior armazenamento em um banco de dados MySQL hospedado no RDS. O fluxo do pipeline é composto por várias etapas integradas: os arquivos são inicialmente recebidos através do SFTP e armazenados em um bucket S3, onde uma função Lambda é acionada para processar e preparar os dados. Em seguida, o AWS Glue realiza a transformação e o carregamento dos dados, que finalmente são inseridos no banco de dados RDS/MySQL.

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

---

# Passo 2. Disparo do Lambda

- Um evento **S3:ObjectCreated** dispara uma função **AWS Lambda**.
- Funções principais do Lambda:
- Validar **metadados** (nome do arquivo, extensão).
- Registrar no **DynamoDB** ou **CloudWatch Logs** que o arquivo chegou.
- Invocar o **AWS Glue Job** passando parâmetros como `bucket`, `key` e `tabela_destino`.

---

# Passo 3. Processamento no Glue

- **Glue Crawler (opcional)**: detecta o schema do CSV e mantém no **Glue Data Catalog**.
- **Glue Job (Spark / PySpark)**:
- Lê o arquivo bruto do S3.
- Realiza **limpeza dos dados** (tipagem, remoção de nulos e duplicados).
- Prepara o **DataFrame** para compatibilidade com o RDS MySQL.
- Insere os dados no RDS/MySQL

---

# Passo 4. Armazenamento no RDS MySQL

- O **Amazon RDS (MySQL)** recebe os dados tratados pelo Glue.
- **Parâmetros de conexão** e senha são armazenados de forma segura no **AWS Secrets Manager**.
- O pipeline garante que os dados sejam **ingestados de forma automatizada, segura e escalável**.

---

