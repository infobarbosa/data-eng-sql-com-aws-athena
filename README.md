# Engenharia de Dados: SQL com AWS Athena

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Descrição do Curso

Este curso tem como objetivo capacitar Analistas de Negócios na execução de consultas analíticas sobre dados armazenados no Amazon S3, utilizando o Amazon Athena como motor de consulta SQL e o AWS Glue Data Catalog como repositório de metadados.

O conteúdo abrange desde os fundamentos da arquitetura serverless de análise de dados na AWS até técnicas estatísticas aplicadas à investigação de anomalias em séries temporais de faturamento.

---

## 2. Público-Alvo

Analistas de Negócios em transição para funções analíticas que operam em ambientes de dados corporativos baseados em nuvem. Não é exigido conhecimento prévio de AWS, mas pressupõe-se familiaridade com conceitos básicos de banco de dados relacional e SQL.

---

## 3. Objetivos de Aprendizagem

Ao concluir este curso, o participante será capaz de:

- Compreender a arquitetura de análise de dados serverless na AWS (S3 + Athena + Glue).
- Definir esquemas de dados externos via DDL para formatos CSV e JSON.
- Executar consultas de projeção, filtragem, junção e agregação sobre dados armazenados no S3.
- Aplicar operações de reestruturação de dados (Pivot/Unpivot) com `CASE WHEN`.
- Utilizar Window Functions para análises de ranking e séries temporais.
- Aplicar métricas estatísticas (desvio padrão e variância) para diagnóstico de instabilidade de faturamento.
- Conduzir uma investigação analítica completa sobre volatilidade de receita.

---

## 4. Competências Desenvolvidas

| Competência | Módulos Relacionados |
|---|---|
| Arquitetura de dados em nuvem | Módulo 1 |
| Definição e gerenciamento de esquemas | Módulo 2 |
| Consultas SQL de filtragem e projeção | Módulo 3 |
| Integração de conjuntos de dados via JOIN | Módulo 4 |
| Análise agregada e sumarização | Módulo 5 |
| Reestruturação tabular de dados | Módulo 6 |
| Análise sequencial com Window Functions | Módulo 7 |
| Diagnóstico estatístico de instabilidade | Módulo 8 |
| Investigação analítica aplicada | Módulo 9 |

---

## 5. Tecnologias Utilizadas

| Tecnologia | Função no Curso |
|---|---|
| **Amazon S3** | Armazenamento persistente dos datasets |
| **Amazon Athena** | Motor de execução de consultas SQL |
| **AWS Glue Data Catalog** | Repositório de metadados (schemas e partições) |
| **SQL (Presto/Trino dialect)** | Linguagem de consulta analítica |

---

## 6. Datasets Utilizados

Todos os laboratórios utilizam os seguintes datasets sintéticos:

| Dataset | Formato | Localização no S3 |
|---|---|---|
| Clientes | JSON (.gz) | `s3://bucket-name/dataset-json-clientes/data/clientes.json.gz` |
| Pedidos | CSV (.gz) | `s3://bucket-name/datasets-csv-pedidos/data/pedidos/` |
| Pagamentos | JSON (.gz) | `s3://bucket-name/dataset-json-pagamentos/data/pagamentos/` |

---

## 7. Estrutura do Curso e Navegação

| Módulo | Título | Link |
|---|---|---|
| 1 | Fundamentos | [modulo-01/README.md](./modulo-01/README.md) |
| 2 | Definição de Dados (DDL) | [modulo-02/README.md](./modulo-02/README.md) |
| 3 | Manipulação e Filtragem | [modulo-03/README.md](./modulo-03/README.md) |
| 4 | Integração de Tabelas (Joins) | [modulo-04/README.md](./modulo-04/README.md) |
| 5 | Agregações Gerenciais | [modulo-05/README.md](./modulo-05/README.md) |
| 6 | Reestruturação de Dados | [modulo-06/README.md](./modulo-06/README.md) |
| 7 | Window Functions | [modulo-07/README.md](./modulo-07/README.md) |
| 8 | Estatística Aplicada | [modulo-08/README.md](./modulo-08/README.md) |
| 9 | Projeto Final | [modulo-09/README.md](./modulo-09/README.md) |

---

## 8. Pré-requisitos Técnicos

- Acesso a uma conta AWS com permissões para Amazon Athena, Amazon S3 e AWS Glue.
- Configuração de um S3 bucket de resultados para o Athena (Output Location).
- Familiaridade com o console AWS ou AWS CLI para execução das consultas.

---

## 9. Modelo de Custo

O Amazon Athena cobra por volume de dados escaneados durante a execução de consultas (**USD 5,00 por TB escaneado**, conforme tabela de preços padrão da região us-east-1). Os laboratórios deste curso foram projetados com datasets compactados (`.gz`) para minimizar o custo operacional durante o aprendizado.
