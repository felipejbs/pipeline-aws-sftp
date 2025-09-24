# Pipeline AWS ingestão de dados com SFTP

<img width="1021" height="371" alt="image" src="https://github.com/user-attachments/assets/b6661866-02f1-444d-85d4-3a12d82b6e08" />

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

# Passo 2 — Disparo do Glue via Lambda

## Objetivo
Garantir que, sempre que um novo arquivo for carregado no **Amazon S3 (Landing Zone)**, um processo seja iniciado automaticamente para tratamento e ingestão dos dados, utilizando **AWS Glue**.

## Como funciona

### Evento S3 (ObjectCreated)
- Sempre que um arquivo é gravado no bucket S3, um evento do tipo **ObjectCreated** é disparado.  
- Esse evento é usado como **gatilho** para a execução da função **AWS Lambda**.  

### AWS Lambda
- A função Lambda é acionada automaticamente pelo evento do S3.  
- Funções principais:  
  - Capturar informações do arquivo (bucket e key).  
  - Validar extensão ou metadados do arquivo (ex.: apenas `.csv`).  
  - Iniciar a execução de um **Glue Job**, passando parâmetros como o caminho do arquivo no S3 e a tabela de destino.  

### Integração com Glue
- A Lambda utiliza o SDK **boto3** para chamar a API `start_job_run` do Glue.  
- Exemplo simplificado em Python:

```python
import boto3
import os

glue = boto3.client("glue")

def lambda_handler(event, context):
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]

        response = glue.start_job_run(
            JobName=os.environ["GLUE_JOB_NAME"],
            Arguments={
                "--s3_input_bucket": bucket,
                "--s3_input_key": key,
                "--db_table": "tabela_destino"
            }
        )
        print("Glue Job iniciado:", response["JobRunId"])
```

---

# Passo 3 — Processamento e Transformação no Glue

## Objetivo
Ler os arquivos **CSV** armazenados no **Amazon S3 (Landing Zone)**, realizar a limpeza, transformação e preparação dos dados, e em seguida carregá-los no banco de dados de destino (**Amazon RDS MySQL**).

## Como funciona

### AWS Glue Job
- É iniciado automaticamente pela função **Lambda** (Passo 2).  
- Executa código em **PySpark** para processar os dados.  
- Principais etapas:  
  1. **Leitura** do arquivo bruto no S3.  
  2. **Limpeza** (remoção de nulos, duplicados, validação de tipos).  
  3. **Transformação** (ajuste de schema, enriquecimento, normalização).  
  4. **Preparação** para escrita no destino.  

### Exemplo simplificado de código PySpark
```python
import sys
from awsglue.context import GlueContext
from pyspark.context import SparkContext
from awsglue.utils import getResolvedOptions
import boto3
import json

# Argumentos recebidos da Lambda
args = getResolvedOptions(sys.argv, ["s3_input_bucket", "s3_input_key", "db_table"])
s3_path = f"s3://{args['s3_input_bucket']}/{args['s3_input_key']}"

# Contexto Spark/Glue
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

# Leitura do CSV
df = spark.read.option("header", True).csv(s3_path)

# Limpeza e transformação (exemplo)
df = df.dropna().dropDuplicates()

# Recupera credenciais do Secrets Manager
secrets_client = boto3.client("secretsmanager")
secret = secrets_client.get_secret_value(SecretId="rds-mysql-secret")
creds = json.loads(secret["SecretString"])

```
---

# Passo 4 — Gravação e Disponibilização no RDS MySQL

## Objetivo
Persistir os dados já **limpos e transformados** no banco de dados **Amazon RDS (MySQL)**, garantindo que fiquem disponíveis para consultas, relatórios e consumo por aplicações.

## Como funciona

### Amazon RDS (MySQL)
- Banco de dados relacional gerenciado pela AWS.  
- Responsável por armazenar os dados processados em tabelas relacionais.  
- Oferece alta disponibilidade e backups automáticos.  

### Escrita dos dados
- Os dados são gravados diretamente no RDS pelo **Glue Job** via bibliotecas Python comuns para conexão com **MySQL**.

### Uso de bibliotecas Python para conexão direta

- `pymysql`  
- `mysql-connector-python`  
- `sqlalchemy`  

#### Exemplo com `pymysql`:
```python
import pymysql

connection = pymysql.connect(
    host="<rds-endpoint>",
    user="<username>",
    password="<password>",
    database="<database>"
)

cursor = connection.cursor()
cursor.execute(
    "INSERT INTO tabela_destino (coluna1, coluna2) VALUES (%s, %s)",
    ("valor1", "valor2")
)
connection.commit()
cursor.close()
connection.close()
```

### Chamada de API

Outra forma de armazenar os dados é chamando uma **API**.  
Essa API atua como intermediária, recebendo os dados processados e inserindo-os no banco de dados.  

#### Vantagens:
- Permite aplicar regras de negócio antes da inserção.  
- Centraliza a lógica de persistência em um único ponto.  
- Facilita auditoria e versionamento da forma como os dados são gravados.  


#### Exemplo de chamada com Python (requests):
```python
import requests

url = "https://api.exemplo.com/ingestao"
payload = {
    "coluna1": "valor1",
    "coluna2": "valor2"
}

response = requests.post(url, json=payload)

if response.status_code == 200:
    print("Dados enviados com sucesso!")
else:
    print("Erro ao enviar dados:", response.text)
```


---

