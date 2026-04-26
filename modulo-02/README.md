# Módulo 2: Definição de Dados (DDL)

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Capacitar o participante na criação de tabelas externas no AWS Glue Data Catalog via instruções DDL executadas pelo Amazon Athena. Ao final do módulo, as três tabelas do curso (`clientes`, `pedidos`, `pagamentos`) estarão registradas e disponíveis para consulta.

---

## 2. Conceito de Tabela Externa

Uma **tabela externa** no Athena é uma definição de schema que aponta para dados preexistentes no S3. O Athena **não move, não copia e não transforma** os dados ao criar a tabela — ele apenas registra os metadados no Glue Data Catalog.

Implicações:

- Excluir a tabela no Athena (`DROP TABLE`) **não exclui** os arquivos no S3.
- Alterar o schema da tabela **não altera** os dados armazenados.
- Os dados são lidos no momento da execução de cada consulta `SELECT`.

---

## 3. Pré-requisitos

Antes de executar os comandos DDL, configure:

1. **Database no Glue Data Catalog:** Crie o banco de dados `db_analytics` via console AWS Glue ou execute no Athena:

```sql
CREATE DATABASE IF NOT EXISTS db_analytics;
```

2. **S3 Output Location:** Configure o bucket de saída do Athena em:
   - Console AWS Athena → Settings → Query result location
   - Exemplo: `s3://bucket-name/athena-results/`

---

## 4. Criação da Tabela: `clientes`

O dataset de clientes está armazenado em formato JSON compactado. Cada linha do arquivo representa um objeto JSON com os seguintes campos:

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | STRING | Identificador único do cliente |
| `nome` | STRING | Nome completo |
| `data_nasc` | STRING | Data de nascimento (formato ISO 8601) |
| `cpf` | STRING | CPF do cliente |
| `email` | STRING | Endereço de e-mail |
| `interesses` | ARRAY\<STRING\> | Lista de categorias de interesse |
| `carteira_investimentos` | MAP\<STRING,DOUBLE\> | Mapa de ativos e valores investidos |

### 4.1 Instrução DDL

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS db_analytics.clientes (
    id                    STRING,
    nome                  STRING,
    data_nasc             STRING,
    cpf                   STRING,
    email                 STRING,
    interesses            ARRAY<STRING>,
    carteira_investimentos MAP<STRING, DOUBLE>
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES (
    'serialization.format' = '1'
)
STORED AS INPUTFORMAT  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT           'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://[O NOME DO SEU BUCKET S3]/clientes/'
TBLPROPERTIES (
    'has_encrypted_data' = 'false',
    'compressionType'    = 'gzip'
);
```

### 4.2 Validação

```sql
-- Verifica se a tabela foi criada com o schema correto
DESCRIBE db_analytics.clientes;

-- Retorna as primeiras 5 linhas para validação de leitura
SELECT *
FROM db_analytics.clientes
LIMIT 5;
```

---

## 5. Criação da Tabela: `pedidos`

O dataset de pedidos está em formato CSV compactado, com delimitador `;`. Os arquivos são organizados mensalmente sob o prefixo `data/pedidos/`.

| Campo | Tipo | Descrição |
|---|---|---|
| `ID_PEDIDO` | STRING | Identificador único do pedido |
| `PRODUTO` | STRING | Descrição do produto |
| `VALOR_UNITARIO` | DOUBLE | Valor unitário do produto |
| `QUANTIDADE` | INT | Quantidade de itens |
| `DATA_CRIACAO` | STRING | Data e hora de criação do pedido |
| `UF` | STRING | Unidade Federativa de origem |
| `ID_CLIENTE` | STRING | Chave estrangeira para a tabela clientes |

### 5.1 Instrução DDL

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS db_analytics.pedidos (
    ID_PEDIDO      STRING,
    PRODUTO        STRING,
    VALOR_UNITARIO DOUBLE,
    QUANTIDADE     INT,
    DATA_CRIACAO   TIMESTAMP,
    UF             STRING,
    ID_CLIENTE     STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
    'serialization.format' = ';',
    'field.delim'          = ';'
)
STORED AS INPUTFORMAT  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT           'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://[O NOME DO SEU BUCKET S3]/pedidos/'
TBLPROPERTIES (
    'has_encrypted_data' = 'false',
    'compressionType'    = 'gzip',
    'skip.header.line.count' = '1'
);
```

> **Observação:** A propriedade `skip.header.line.count = '1'` instrui o Athena a ignorar a primeira linha de cada arquivo CSV, que contém os nomes das colunas.

### 5.2 Validação

```sql
DESCRIBE db_analytics.pedidos;

SELECT *
FROM db_analytics.pedidos
LIMIT 5;
```

---

## 6. Criação da Tabela: `pagamentos`

O dataset de pagamentos está em formato JSON compactado. O campo `avaliacao_fraude` é um tipo estruturado (`STRUCT`) contendo dois subcampos.

| Campo | Tipo | Descrição |
|---|---|---|
| `id_pedido` | STRING | Referência ao pedido correspondente |
| `forma_pagamento` | STRING | Modalidade de pagamento |
| `valor_pagamento` | DOUBLE | Valor total processado |
| `status` | BOOLEAN | Indica se o pagamento foi aprovado |
| `data_processamento` | TIMESTAMP | Data e hora do processamento |
| `avaliacao_fraude` | STRUCT | Resultado da avaliação de risco |
| `avaliacao_fraude.fraude` | BOOLEAN | Indica detecção de fraude |
| `avaliacao_fraude.score` | DOUBLE | Score de risco (0.0 a 1.0) |

### 6.1 Instrução DDL

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS db_analytics.pagamentos (
    id_pedido          STRING,
    forma_pagamento    STRING,
    valor_pagamento    DOUBLE,
    status             BOOLEAN,
    data_processamento TIMESTAMP,
    avaliacao_fraude   STRUCT<
        fraude : BOOLEAN,
        score  : DOUBLE
    >
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES (
    'serialization.format' = '1'
)
STORED AS INPUTFORMAT  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT           'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://[O NOME DO SEU BUCKET S3]/pagamentos/'
TBLPROPERTIES (
    'has_encrypted_data' = 'false',
    'compressionType'    = 'gzip'
);
```

### 6.2 Acesso a Campos de STRUCT

Para consultar subcampos de um tipo `STRUCT`, utilize a notação de ponto:

```sql
SELECT
    id_pedido,
    avaliacao_fraude.fraude AS detectou_fraude,
    avaliacao_fraude.score  AS score_risco
FROM db_analytics.pagamentos
LIMIT 10;
```

---

## 7. Verificação do Estado do Catálogo

```sql
-- Lista todas as tabelas do database
SHOW TABLES IN db_analytics;

-- Exibe o DDL completo de uma tabela (útil para auditoria de schema)
SHOW CREATE TABLE db_analytics.clientes;
SHOW CREATE TABLE db_analytics.pedidos;
SHOW CREATE TABLE db_analytics.pagamentos;
```

---

## 8. Operações de Manutenção

### 8.1 Remoção de Tabela

```sql
-- Remove a definição da tabela do catálogo (não exclui dados no S3)
DROP TABLE IF EXISTS db_analytics.pedidos;
```

### 8.2 Adição de Coluna

```sql
-- Adiciona uma coluna ao schema existente
ALTER TABLE db_analytics.pedidos
ADD COLUMNS (canal_venda STRING);
```

> **Atenção:** A adição de coluna via `ALTER TABLE` apenas atualiza o schema no catálogo. Se os arquivos no S3 não contiverem esse campo, o Athena retornará `NULL` para a nova coluna.

---

## 9. Resumo do Módulo

| Tabela | Formato | SerDe | Tipo Especial |
|---|---|---|---|
| `clientes` | JSON (.gz) | JsonSerDe | ARRAY, MAP |
| `pedidos` | CSV (.gz) | LazySimpleSerDe | — |
| `pagamentos` | JSON (.gz) | JsonSerDe | STRUCT |

---

[← Módulo 1](../modulo-01/README.md) | [Próximo: Módulo 3 →](../modulo-03/README.md)
