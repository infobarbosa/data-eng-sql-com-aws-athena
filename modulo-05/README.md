# Módulo 5: Agregações Gerenciais

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Capacitar o participante na elaboração de consultas de sumarização de dados utilizando a cláusula `GROUP BY` e as funções de agregação do Amazon Athena. O foco é a construção de visões gerenciais que respondam a perguntas de negócio sobre volume de vendas, faturamento e comportamento de pagamentos.

---

## 2. Funções de Agregação

As funções de agregação operam sobre um conjunto de registros e retornam um único valor. Elas são aplicadas **após** o agrupamento definido pela cláusula `GROUP BY`.

| Função | Descrição |
|---|---|
| `COUNT(*)` | Contagem de registros (incluindo NULLs) |
| `COUNT(coluna)` | Contagem de valores não nulos |
| `COUNT(DISTINCT coluna)` | Contagem de valores distintos |
| `SUM(coluna)` | Soma dos valores |
| `AVG(coluna)` | Média aritmética |
| `MIN(coluna)` | Valor mínimo |
| `MAX(coluna)` | Valor máximo |

---

## 3. Cláusula GROUP BY

### 3.1 Funcionamento

A cláusula `GROUP BY` divide os registros da tabela em grupos, onde cada grupo possui o mesmo valor nas colunas especificadas. As funções de agregação são calculadas **por grupo**.

**Regra fundamental:** Toda coluna presente no `SELECT` que **não** for uma função de agregação **deve** estar presente no `GROUP BY`.

### 3.2 Agregações sobre Pedidos

**Faturamento por estado:**

```sql
SELECT
    UF,
    COUNT(ID_PEDIDO)                        AS total_pedidos,
    SUM(VALOR_UNITARIO * QUANTIDADE)        AS faturamento_total,
    AVG(VALOR_UNITARIO * QUANTIDADE)        AS ticket_medio,
    MIN(VALOR_UNITARIO * QUANTIDADE)        AS menor_pedido,
    MAX(VALOR_UNITARIO * QUANTIDADE)        AS maior_pedido
FROM db_analytics.pedidos
GROUP BY UF
ORDER BY faturamento_total DESC;
```

**Volume de pedidos por produto:**

```sql
SELECT
    PRODUTO,
    COUNT(ID_PEDIDO)                 AS total_pedidos,
    SUM(QUANTIDADE)                  AS unidades_vendidas,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS receita_total,
    AVG(VALOR_UNITARIO)              AS preco_medio_unitario
FROM db_analytics.pedidos
GROUP BY PRODUTO
ORDER BY receita_total DESC;
```

**Agrupamento por múltiplas colunas:**

```sql
-- Faturamento por estado e por produto
SELECT
    UF,
    PRODUTO,
    COUNT(ID_PEDIDO)                 AS total_pedidos,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento
FROM db_analytics.pedidos
GROUP BY UF, PRODUTO
ORDER BY UF, faturamento DESC;
```

---

## 4. Filtragem de Grupos: Cláusula HAVING

A cláusula `HAVING` filtra **grupos** após a agregação. Ela opera sobre o resultado do `GROUP BY`, enquanto a cláusula `WHERE` opera sobre registros individuais antes do agrupamento.

**Ordem de execução:**

```
FROM → JOIN → WHERE → GROUP BY → funções de agregação → HAVING → SELECT → ORDER BY → LIMIT
```

```sql
-- Estados com faturamento total superior a R$ 500.000,00
SELECT
    UF,
    COUNT(ID_PEDIDO)                 AS total_pedidos,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento_total
FROM db_analytics.pedidos
GROUP BY UF
HAVING SUM(VALOR_UNITARIO * QUANTIDADE) > 500000.00
ORDER BY faturamento_total DESC;
```

```sql
-- Produtos vendidos em mais de 100 pedidos
SELECT
    PRODUTO,
    COUNT(ID_PEDIDO) AS total_pedidos
FROM db_analytics.pedidos
GROUP BY PRODUTO
HAVING COUNT(ID_PEDIDO) > 100
ORDER BY total_pedidos DESC;
```

---

## 5. Agregações sobre Pagamentos

```sql
-- Volume e valor por forma de pagamento
SELECT
    forma_pagamento,
    COUNT(id_pedido)         AS total_transacoes,
    SUM(valor_pagamento)     AS volume_total,
    AVG(valor_pagamento)     AS ticket_medio,
    AVG(avaliacao_fraude.score) AS score_fraude_medio
FROM db_analytics.pagamentos
WHERE status = true
GROUP BY forma_pagamento
ORDER BY volume_total DESC;
```

```sql
-- Taxa de aprovação por forma de pagamento
SELECT
    forma_pagamento,
    COUNT(id_pedido)                                          AS total,
    SUM(CASE WHEN status = true THEN 1 ELSE 0 END)           AS aprovados,
    SUM(CASE WHEN status = false THEN 1 ELSE 0 END)          AS reprovados,
    ROUND(
        100.0 * SUM(CASE WHEN status = true THEN 1 ELSE 0 END) / COUNT(id_pedido),
        2
    )                                                          AS taxa_aprovacao_pct
FROM db_analytics.pagamentos
GROUP BY forma_pagamento
ORDER BY taxa_aprovacao_pct DESC;
```

---

## 6. Agregações Temporais

Para análises temporais, o Athena oferece funções de extração de componentes de data:

```sql
-- Faturamento mensal de pedidos em 2025
SELECT
    YEAR(DATE(DATA_CRIACAO))  AS ano,
    MONTH(DATE(DATA_CRIACAO)) AS mes,
    COUNT(ID_PEDIDO)                        AS total_pedidos,
    SUM(VALOR_UNITARIO * QUANTIDADE)        AS faturamento_mensal
FROM db_analytics.pedidos
WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
GROUP BY
    YEAR(DATE(DATA_CRIACAO)),
    MONTH(DATE(DATA_CRIACAO))
ORDER BY ano, mes;
```

```sql
-- Faturamento semanal
SELECT
    DATE_TRUNC('week', DATE(DATA_CRIACAO)) AS semana_inicio,
    COUNT(ID_PEDIDO)                        AS total_pedidos,
    SUM(VALOR_UNITARIO * QUANTIDADE)        AS faturamento_semanal
FROM db_analytics.pedidos
WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
GROUP BY DATE_TRUNC('week', DATE(DATA_CRIACAO))
ORDER BY semana_inicio;
```

---

## 7. Agregações com JOIN

```sql
-- Faturamento por estado com taxa de aprovação de pagamentos
SELECT
    p.UF,
    COUNT(DISTINCT p.ID_PEDIDO)                               AS total_pedidos,
    SUM(p.VALOR_UNITARIO * p.QUANTIDADE)                      AS faturamento_bruto,
    SUM(CASE WHEN pg.status = true
             THEN p.VALOR_UNITARIO * p.QUANTIDADE
             ELSE 0 END)                                       AS faturamento_aprovado,
    ROUND(
        100.0 * SUM(CASE WHEN pg.status = true THEN 1 ELSE 0 END)
              / COUNT(p.ID_PEDIDO),
        2
    )                                                          AS taxa_aprovacao_pct
FROM db_analytics.pedidos p
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido
GROUP BY p.UF
ORDER BY faturamento_aprovado DESC;
```

---

## 8. Laboratório Prático

**Exercício 1:** Calcule o faturamento total, ticket médio e número de pedidos para cada UF, considerando apenas pedidos com quantidade superior a 1. Ordene por faturamento decrescente.

```sql
SELECT
    UF,
    COUNT(ID_PEDIDO)                 AS total_pedidos,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento_total,
    AVG(VALOR_UNITARIO * QUANTIDADE) AS ticket_medio
FROM db_analytics.pedidos
WHERE QUANTIDADE > 1
GROUP BY UF
ORDER BY faturamento_total DESC;
```

**Exercício 2:** Identifique os 5 produtos com maior receita total em 2025 e exiba o número de estados distintos em que foram vendidos.

```sql
SELECT
    PRODUTO,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS receita_total,
    COUNT(DISTINCT UF)               AS estados_distintos,
    COUNT(ID_PEDIDO)                 AS total_pedidos
FROM db_analytics.pedidos
WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
GROUP BY PRODUTO
ORDER BY receita_total DESC
LIMIT 5;
```

**Exercício 3:** Liste as formas de pagamento com mais de 50 transações aprovadas e score médio de fraude superior a 0.3.

```sql
SELECT
    forma_pagamento,
    COUNT(id_pedido)                AS transacoes_aprovadas,
    AVG(avaliacao_fraude.score)     AS score_fraude_medio,
    SUM(valor_pagamento)            AS volume_aprovado
FROM db_analytics.pagamentos
WHERE status = true
GROUP BY forma_pagamento
HAVING COUNT(id_pedido) > 50
   AND AVG(avaliacao_fraude.score) > 0.3
ORDER BY volume_aprovado DESC;
```

---

## 9. Resumo do Módulo

| Cláusula | Momento de Execução | Finalidade |
|---|---|---|
| `WHERE` | Antes do `GROUP BY` | Filtra registros individuais |
| `GROUP BY` | Após `WHERE` | Agrupa registros |
| Funções de agregação | Após `GROUP BY` | Calculam métricas por grupo |
| `HAVING` | Após agregação | Filtra grupos |
| `ORDER BY` | Após `HAVING` | Ordena o resultado final |

---

[← Módulo 4](../modulo-04/README.md) | [Próximo: Módulo 6 →](../modulo-06/README.md)
