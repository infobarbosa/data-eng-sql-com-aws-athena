# Módulo 1: Fundamentos

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Apresentar a arquitetura de análise de dados serverless da AWS, composta por três serviços interdependentes: **Amazon S3**, **Amazon Athena** e **AWS Glue Data Catalog**. O participante compreenderá o fluxo completo de um dado — desde o armazenamento bruto no S3 até a execução de uma consulta SQL — e o impacto financeiro de cada operação.

---

## 2. Arquitetura de Referência

```
┌──────────────────────┐
│      Amazon S3       │  ← Armazenamento persistente de objetos
│  (dados brutos .gz)  │
└─────────┬────────────┘
          │ leitura dos arquivos
          ▼
┌──────────────────────┐     ┌───────────────────────────────┐
│    Amazon Athena     │────▶│   AWS Glue Data Catalog        │
│  (motor SQL/Presto)  │     │ (schemas, tipos, partições)    │
└─────────┬────────────┘     └───────────────────────────────┘
          │ resultado
          ▼
┌──────────────────────┐
│  S3 Output Location  │  ← Arquivos CSV com resultados
└──────────────────────┘
```

**Fluxo de execução de uma consulta:**

1. O analista submete uma instrução SQL via console AWS, JDBC/ODBC ou API.
2. O Athena consulta o **Glue Data Catalog** para obter o schema da tabela (tipos, localização, formato, SerDe).
3. Com base no schema, o Athena localiza e lê os arquivos no **Amazon S3**.
4. O resultado é gravado no bucket S3 configurado como **Output Location**.
5. O volume de dados lido no passo 3 determina o custo da operação.

---

## 3. Amazon S3

### 3.1 Definição Técnica

O **Amazon Simple Storage Service (S3)** é um serviço de armazenamento de objetos com durabilidade de 99,999999999% (11 noves). No contexto deste curso, o S3 funciona como camada de persistência de todos os datasets.

Um **objeto** no S3 é qualquer arquivo armazenado, identificado por uma chave única dentro de um bucket. A estrutura de pastas visível no console é uma convenção baseada em prefixos de chave — o S3 não possui hierarquia de diretórios nativa.

### 3.2 Estrutura dos Datasets

| Dataset | Formato | Localização |
|---|---|---|
| Clientes | JSON (.gz) | `s3://bucket-name/dataset-json-clientes/data/clientes.json.gz` |
| Pedidos | CSV (.gz) | `s3://bucket-name/datasets-csv-pedidos/data/pedidos/` |
| Pagamentos | JSON (.gz) | `s3://bucket-name/dataset-json-pagamentos/data/pagamentos/` |

A compactação com `gzip` reduz o volume de dados armazenados e escaneados pelo Athena, com impacto direto no custo por consulta.

---

## 4. Amazon Athena

### 4.1 Definição Técnica

O **Amazon Athena** é um serviço de consulta SQL serverless baseado no motor **Presto/Trino**. Ele permite executar consultas SQL diretamente sobre dados armazenados no S3, sem necessidade de provisionar infraestrutura.

O Athena é um **Query Engine**, não um banco de dados. Ele não armazena dados: lê arquivos no S3, processa a consulta de forma distribuída e descarta os dados intermediários após a conclusão.

### 4.2 Características Operacionais

| Característica | Detalhe |
|---|---|
| Modelo de execução | Serverless — sem provisionamento |
| Motor de consulta | Presto/Trino (SQL ANSI com extensões) |
| Formatos suportados | CSV, JSON, Parquet, ORC, Avro |
| Compressão suportada | GZIP, SNAPPY, ZSTD, BZIP2 |
| Unidade de cobrança | Volume de dados escaneados por consulta |
| Output | Arquivos CSV gravados no S3 Output Location |

### 4.3 Fases Internas de Processamento

1. **Parsing e validação** da instrução SQL.
2. **Planejamento de execução**: determinação de quais arquivos serão lidos e se há partições a filtrar.
3. **Execução distribuída**: leitura paralela dos arquivos S3 em múltiplos workers.
4. **Agregação** dos resultados parciais.
5. **Gravação** do resultado final no S3 Output Location.

---

## 5. AWS Glue Data Catalog

### 5.1 Definição Técnica

O **AWS Glue Data Catalog** é o repositório centralizado de metadados da AWS. Ele armazena a definição formal de cada tabela: nome das colunas, tipos de dados, localização dos arquivos no S3, formato, delimitador (para CSV) e SerDe.

O Athena **depende obrigatoriamente** do Glue Data Catalog. Sem um schema registrado, o Athena não tem como interpretar os bytes lidos do S3.

### 5.2 Conceito de SerDe

**SerDe** (Serializer/Deserializer) é o componente responsável por converter o formato de um arquivo em colunas tipadas processáveis pelo Athena.

| Formato | SerDe |
|---|---|
| CSV / TSV | `LazySimpleSerDe` |
| JSON (uma linha por registro) | `org.apache.hive.hcatalog.data.JsonSerDe` |
| Parquet | `ParquetHiveSerDe` |

### 5.3 Estrutura Lógica do Catálogo

```
Glue Data Catalog
└── Database: db_analytics
    ├── Table: clientes
    │   ├── Columns: id, nome, data_nasc, cpf, email, interesses, carteira_investimentos
    │   ├── Location: s3://bucket-name/dataset-json-clientes/data/
    │   └── SerDe: JsonSerDe
    ├── Table: pedidos
    │   ├── Columns: ID_PEDIDO, PRODUTO, VALOR_UNITARIO, QUANTIDADE, DATA_CRIACAO, UF, ID_CLIENTE
    │   ├── Location: s3://bucket-name/datasets-csv-pedidos/data/pedidos/
    │   └── SerDe: LazySimpleSerDe (delimitador: ;)
    └── Table: pagamentos
        ├── Columns: id_pedido, forma_pagamento, valor_pagamento, status, data_processamento, avaliacao_fraude
        ├── Location: s3://bucket-name/dataset-json-pagamentos/data/pagamentos/
        └── SerDe: JsonSerDe
```

---

## 6. Modelo de Custo

### 6.1 Estrutura de Preço

O Athena adota o modelo **pay-per-query**, calculado pelo **volume de dados escaneados**:

> **Custo padrão (us-east-1):** USD 5,00 por TB de dados escaneados.

O custo mínimo por consulta corresponde a **10 MB escaneados**.

### 6.2 Fatores que Influenciam o Custo

| Fator | Impacto |
|---|---|
| Volume total do arquivo | Maior arquivo → maior custo |
| Compressão dos dados | Arquivos `.gz` reduzem significativamente o escaneamento |
| Uso de partições | Filtros sobre colunas de partição limitam os arquivos lidos |
| Formato colunar (Parquet/ORC) | Permite leitura seletiva de colunas |

### 6.3 Operações Sem Custo de Escaneamento

- `DDL` statements (`CREATE TABLE`, `DROP TABLE`, `ALTER TABLE`).
- Consultas com falha de sintaxe canceladas antes da execução.
- Consultas canceladas antes do início do escaneamento.

---

## 7. Resumo do Módulo

| Serviço | Função | Modelo de Cobrança |
|---|---|---|
| Amazon S3 | Armazenamento dos dados brutos | Por GB armazenado |
| Amazon Athena | Execução de consultas SQL | Por TB de dados escaneados |
| AWS Glue Data Catalog | Repositório de metadados | Por objetos e requisições de API |

---

[← Início](../README.md) | [Próximo: Módulo 2 →](../modulo-02/README.md)
