# Módulo 3: Manipulação e Filtragem

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Desenvolver a capacidade de executar consultas de **projeção de colunas** e **filtragem de registros** no Amazon Athena. O participante aprenderá a controlar quais dados são retornados por uma consulta e, criticamente, o impacto dessas decisões no volume de dados escaneados e no custo operacional.

---

## 2. Projeção de Colunas

### 2.1 Conceito

**Projeção de colunas** é a operação de selecionar um subconjunto das colunas disponíveis em uma tabela. No SQL, é implementada pela cláusula `SELECT`.

Em formatos de armazenamento **orientados a linha** (CSV, JSON), a projeção de colunas **não reduz** o volume de dados escaneados — o Athena lê o registro completo e descarta as colunas não solicitadas após a leitura. Em formatos **colunares** (Parquet, ORC), a projeção reduz o escaneamento, pois apenas as colunas necessárias são lidas do disco.

### 2.2 Seleção de Todas as Colunas

```sql
-- Retorna todos os campos de todos os registros
-- Custo: escaneia o arquivo completo
SELECT *
FROM db_analytics.pedidos;
```

> **Atenção:** O uso de `SELECT *` em ambiente de produção deve ser evitado quando o objetivo é apenas um subconjunto de colunas. Mesmo em formato CSV, o hábito de projetar apenas as colunas necessárias é considerado boa prática para legibilidade e manutenibilidade.

### 2.3 Projeção Seletiva

```sql
-- Retorna apenas as colunas de interesse analítico
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    QUANTIDADE,
    UF
FROM db_analytics.pedidos;
```

### 2.4 Colunas Calculadas

O Athena permite criar colunas derivadas diretamente na projeção:

```sql
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    QUANTIDADE,
    VALOR_UNITARIO * QUANTIDADE AS VALOR_TOTAL,
    UF,
    DATA_CRIACAO
FROM db_analytics.pedidos;
```

### 2.5 Renomeação de Colunas com Alias

```sql
SELECT
    ID_PEDIDO      AS pedido_id,
    PRODUTO        AS descricao_produto,
    VALOR_UNITARIO AS preco_unitario,
    QUANTIDADE     AS qtd,
    UF             AS estado
FROM db_analytics.pedidos;
```

---

## 3. Filtragem de Registros: Cláusula WHERE

### 3.1 Conceito

A cláusula `WHERE` filtra registros **após** a leitura do arquivo, aplicando uma condição booleana a cada linha. Registros que não satisfazem a condição são descartados do resultado.

> **Importante:** Em tabelas CSV/JSON sem particionamento, a cláusula `WHERE` **não reduz o volume escaneado**. O Athena lê todos os arquivos e aplica o filtro em memória. A redução do custo por filtragem só ocorre com o uso de **partições** (abordado em módulos avançados).

### 3.2 Filtros Simples

```sql
-- Pedidos do estado de São Paulo
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    QUANTIDADE,
    DATA_CRIACAO
FROM db_analytics.pedidos
WHERE UF = 'SP';
```

```sql
-- Pedidos com valor unitário acima de R$ 500,00
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    UF
FROM db_analytics.pedidos
WHERE VALOR_UNITARIO > 500.00;
```

### 3.3 Filtros Combinados com AND / OR

```sql
-- Pedidos de SP com valor unitário acima de R$ 200,00
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    QUANTIDADE
FROM db_analytics.pedidos
WHERE UF = 'SP'
  AND VALOR_UNITARIO > 200.00;
```

```sql
-- Pedidos de SP ou RJ
SELECT
    ID_PEDIDO,
    PRODUTO,
    UF,
    DATA_CRIACAO
FROM db_analytics.pedidos
WHERE UF = 'SP'
   OR UF = 'RJ';
```

### 3.4 Filtro com IN

O operador `IN` é funcionalmente equivalente a múltiplos `OR`, porém com sintaxe mais legível:

```sql
SELECT
    ID_PEDIDO,
    PRODUTO,
    UF,
    VALOR_UNITARIO
FROM db_analytics.pedidos
WHERE UF IN ('SP', 'RJ', 'MG', 'RS');
```

### 3.5 Filtro com BETWEEN

```sql
-- Pedidos com valor unitário entre R$ 100,00 e R$ 500,00 (inclusive)
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO
FROM db_analytics.pedidos
WHERE VALOR_UNITARIO BETWEEN 100.00 AND 500.00;
```

### 3.6 Filtro sobre Texto com LIKE

```sql
-- Produtos cujo nome começa com "Notebook"
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO
FROM db_analytics.pedidos
WHERE PRODUTO LIKE 'Notebook%';
```

```sql
-- Produtos que contêm a palavra "Pro" em qualquer posição
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO
FROM db_analytics.pedidos
WHERE PRODUTO LIKE '%Pro%';
```

### 3.7 Filtro sobre Datas

O campo `DATA_CRIACAO` está armazenado como `STRING`. Para comparações de data, é necessário converter para o tipo `DATE` ou `TIMESTAMP`:

```sql
-- Pedidos criados a partir de 2025-03-01
SELECT
    ID_PEDIDO,
    PRODUTO,
    DATA_CRIACAO,
    UF
FROM db_analytics.pedidos
WHERE DATE(DATA_CRIACAO) >= DATE('2025-03-01');
```

```sql
-- Pedidos do mês de janeiro de 2025
SELECT
    ID_PEDIDO,
    PRODUTO,
    DATA_CRIACAO
FROM db_analytics.pedidos
WHERE DATE(DATA_CRIACAO) BETWEEN DATE('2025-01-01') AND DATE('2025-01-31');
```

### 3.8 Filtro sobre Booleanos (tabela pagamentos)

```sql
-- Pagamentos aprovados (status = true)
SELECT
    id_pedido,
    forma_pagamento,
    valor_pagamento,
    data_processamento
FROM db_analytics.pagamentos
WHERE status = true;
```

```sql
-- Pagamentos com suspeita de fraude
SELECT
    id_pedido,
    forma_pagamento,
    valor_pagamento,
    avaliacao_fraude.score AS score_fraude
FROM db_analytics.pagamentos
WHERE avaliacao_fraude.fraude = true;
```

### 3.9 Filtro sobre Valores Nulos

```sql
-- Pedidos sem UF informada
SELECT
    ID_PEDIDO,
    PRODUTO,
    UF
FROM db_analytics.pedidos
WHERE UF IS NULL;
```

```sql
-- Pedidos com UF informada
SELECT
    ID_PEDIDO,
    PRODUTO,
    UF
FROM db_analytics.pedidos
WHERE UF IS NOT NULL;
```

---

## 4. Controle de Volume de Resultado

### 4.1 Cláusula LIMIT

```sql
-- Retorna apenas os 10 primeiros registros
-- Útil para inspeção de schema e validação de dados
SELECT *
FROM db_analytics.clientes
LIMIT 10;
```

> **Atenção:** O `LIMIT` **não reduz o volume de dados escaneados** no Athena. O motor lê os arquivos completos e trunca o resultado ao final. Para reduzir custo, utilize partições e filtros sobre colunas de partição.

### 4.2 Eliminação de Duplicatas com DISTINCT

```sql
-- Lista todos os estados presentes no dataset de pedidos (sem repetição)
SELECT DISTINCT UF
FROM db_analytics.pedidos
ORDER BY UF;
```

```sql
-- Lista todas as formas de pagamento únicas
SELECT DISTINCT forma_pagamento
FROM db_analytics.pagamentos
ORDER BY forma_pagamento;
```

---

## 5. Ordenação de Resultados

```sql
-- Pedidos ordenados por valor unitário decrescente
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    UF
FROM db_analytics.pedidos
ORDER BY VALOR_UNITARIO DESC
LIMIT 20;
```

```sql
-- Clientes ordenados por nome alfabeticamente
SELECT
    id,
    nome,
    email
FROM db_analytics.clientes
ORDER BY nome ASC
LIMIT 50;
```

---

## 6. Laboratório Prático

Execute as consultas abaixo e registre os resultados:

**Exercício 1:** Selecione `ID_PEDIDO`, `PRODUTO`, `VALOR_UNITARIO` e `QUANTIDADE` dos pedidos do estado `'SC'` com quantidade superior a 3 unidades.

```sql
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    QUANTIDADE
FROM db_analytics.pedidos
WHERE UF = 'SC'
  AND QUANTIDADE > 3;
```

**Exercício 2:** Liste os pedidos cujo valor total (`VALOR_UNITARIO * QUANTIDADE`) excede R$ 1.000,00, exibindo o valor calculado como `VALOR_TOTAL`.

```sql
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO,
    QUANTIDADE,
    VALOR_UNITARIO * QUANTIDADE AS VALOR_TOTAL,
    UF
FROM db_analytics.pedidos
WHERE VALOR_UNITARIO * QUANTIDADE > 1000.00
ORDER BY VALOR_TOTAL DESC;
```

**Exercício 3:** Liste os pagamentos aprovados (`status = true`) com score de fraude superior a 0.7.

```sql
SELECT
    id_pedido,
    forma_pagamento,
    valor_pagamento,
    avaliacao_fraude.score AS score_fraude
FROM db_analytics.pagamentos
WHERE status = true
  AND avaliacao_fraude.score > 0.7
ORDER BY score_fraude DESC;
```

---

## 7. Resumo do Módulo

| Operação | Cláusula SQL | Impacto no Custo |
|---|---|---|
| Projeção de colunas | `SELECT col1, col2` | Sem impacto em CSV/JSON |
| Filtragem de linhas | `WHERE condição` | Sem impacto em tabelas não particionadas |
| Limitação de resultado | `LIMIT n` | Sem impacto no escaneamento |
| Eliminação de duplicatas | `DISTINCT` | Sem impacto no escaneamento |
| Ordenação | `ORDER BY` | Sem impacto no escaneamento |

---

[← Módulo 2](../modulo-02/README.md) | [Próximo: Módulo 4 →](../modulo-04/README.md)
