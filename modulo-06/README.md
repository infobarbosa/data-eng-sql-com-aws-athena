# Módulo 6: Reestruturação de Dados

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Capacitar o participante na aplicação de operações de **Pivot** e **Unpivot** para reestruturação do layout tabular dos dados. O módulo utiliza a expressão `CASE WHEN` como mecanismo central dessas transformações, dado que o Athena (Presto/Trino) não possui as cláusulas `PIVOT` e `UNPIVOT` nativas do SQL Server ou Oracle.

---

## 2. Operação de Pivot

### 2.1 Definição

**Pivot** é a operação que transforma **valores de uma coluna em colunas distintas** do resultado, agrupando e agregando os dados correspondentes. O resultado passa de um formato **longo** (múltiplos registros por entidade) para um formato **largo** (um registro por entidade com colunas para cada categoria).

**Estrutura de dados antes do Pivot (formato longo):**

| UF | forma_pagamento | valor_pagamento |
|---|---|---|
| SP | CREDITO | 500.00 |
| SP | DEBITO | 200.00 |
| RJ | CREDITO | 300.00 |
| RJ | PIX | 150.00 |

**Estrutura de dados após o Pivot (formato largo):**

| UF | CREDITO | DEBITO | PIX |
|---|---|---|---|
| SP | 500.00 | 200.00 | 0 |
| RJ | 300.00 | 0 | 150.00 |

### 2.2 Implementação com CASE WHEN

A lógica do Pivot com `CASE WHEN` consiste em criar uma coluna para cada valor que se deseja pivotar, usando a expressão condicional para selecionar apenas o valor correspondente e `NULL` (ou 0) para os demais. A função de agregação (`SUM`, `MAX`, `COUNT`) consolida os valores por grupo.

```sql
-- Faturamento mensal por forma de pagamento (colunas por mês)
SELECT
    MONTH(DATE(pg.data_processamento))                              AS mes,
    SUM(CASE WHEN pg.forma_pagamento = 'CREDITO'
             THEN pg.valor_pagamento ELSE 0 END)                   AS credito,
    SUM(CASE WHEN pg.forma_pagamento = 'DEBITO'
             THEN pg.valor_pagamento ELSE 0 END)                   AS debito,
    SUM(CASE WHEN pg.forma_pagamento = 'PIX'
             THEN pg.valor_pagamento ELSE 0 END)                   AS pix,
    SUM(CASE WHEN pg.forma_pagamento = 'BOLETO'
             THEN pg.valor_pagamento ELSE 0 END)                   AS boleto,
    SUM(pg.valor_pagamento)                                         AS total_geral
FROM db_analytics.pagamentos pg
WHERE YEAR(DATE(pg.data_processamento)) = 2025
  AND pg.status = true
GROUP BY MONTH(DATE(pg.data_processamento))
ORDER BY mes;
```

### 2.3 Pivot por Estado (UF)

```sql
-- Número de pedidos por produto em cada UF (top 4 UFs)
SELECT
    PRODUTO,
    SUM(CASE WHEN UF = 'SP' THEN 1 ELSE 0 END) AS SP,
    SUM(CASE WHEN UF = 'RJ' THEN 1 ELSE 0 END) AS RJ,
    SUM(CASE WHEN UF = 'MG' THEN 1 ELSE 0 END) AS MG,
    SUM(CASE WHEN UF = 'RS' THEN 1 ELSE 0 END) AS RS,
    COUNT(ID_PEDIDO)                            AS total_geral
FROM db_analytics.pedidos
GROUP BY PRODUTO
ORDER BY total_geral DESC;
```

### 2.4 Pivot com Dados Cruzados: Pedidos e Pagamentos

```sql
-- Faturamento por estado e status de pagamento (aprovado vs reprovado)
SELECT
    p.UF,
    SUM(CASE WHEN pg.status = true
             THEN p.VALOR_UNITARIO * p.QUANTIDADE ELSE 0 END) AS faturamento_aprovado,
    SUM(CASE WHEN pg.status = false
             THEN p.VALOR_UNITARIO * p.QUANTIDADE ELSE 0 END) AS faturamento_reprovado,
    COUNT(CASE WHEN pg.status = true  THEN 1 END)             AS pedidos_aprovados,
    COUNT(CASE WHEN pg.status = false THEN 1 END)             AS pedidos_reprovados
FROM db_analytics.pedidos p
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido
GROUP BY p.UF
ORDER BY faturamento_aprovado DESC;
```

---

## 3. Operação de Unpivot

### 3.1 Definição

**Unpivot** é a operação inversa do Pivot. Transforma **colunas em valores de uma única coluna**, convertendo o formato **largo** em formato **longo**. É utilizado quando o dado chega em formato de relatório (colunas por período, por categoria) e precisa ser normalizado para análises subsequentes.

**Estrutura antes do Unpivot (formato largo):**

| ID_PEDIDO | jan | fev | mar |
|---|---|---|---|
| P001 | 500 | 300 | 700 |
| P002 | 200 | 0 | 450 |

**Estrutura após o Unpivot (formato longo):**

| ID_PEDIDO | mes | valor |
|---|---|---|
| P001 | jan | 500 |
| P001 | fev | 300 |
| P001 | mar | 700 |
| P002 | jan | 200 |
| P002 | fev | 0 |
| P002 | mar | 450 |

### 3.2 Implementação com CROSS JOIN e VALUES

No Athena/Presto, o Unpivot pode ser implementado via `CROSS JOIN UNNEST` ou com subconsultas usando `UNION ALL`:

**Abordagem com UNION ALL:**

```sql
-- Desnormaliza dados de avaliação de fraude em linhas individuais
-- (exemplo conceitual com CTE)
WITH pagamentos_consolidados AS (
    SELECT
        id_pedido,
        forma_pagamento,
        valor_pagamento,
        avaliacao_fraude.fraude AS detectou_fraude,
        avaliacao_fraude.score  AS score_fraude
    FROM db_analytics.pagamentos
    WHERE status = true
)
SELECT id_pedido, 'valor_pagamento' AS metrica, CAST(valor_pagamento AS VARCHAR) AS valor
FROM pagamentos_consolidados
UNION ALL
SELECT id_pedido, 'score_fraude', CAST(score_fraude AS VARCHAR)
FROM pagamentos_consolidados
ORDER BY id_pedido, metrica;
```

### 3.3 Unpivot com CROSS JOIN e Tabela de Valores

```sql
-- Técnica de Unpivot usando CROSS JOIN com tabela inline
WITH pedidos_mensais AS (
    SELECT
        UF,
        SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 1
                 THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END) AS jan,
        SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 2
                 THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END) AS fev,
        SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 3
                 THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END) AS mar
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY UF
)
SELECT
    pm.UF,
    meses.mes,
    CASE meses.mes
        WHEN 'jan' THEN pm.jan
        WHEN 'fev' THEN pm.fev
        WHEN 'mar' THEN pm.mar
    END AS faturamento
FROM pedidos_mensais pm
CROSS JOIN (
    VALUES ('jan'), ('fev'), ('mar')
) AS meses(mes)
ORDER BY pm.UF, meses.mes;
```

---

## 4. Expressão CASE WHEN: Referência Técnica

A expressão `CASE WHEN` é o mecanismo central das operações de Pivot e Unpivot. Ela é uma expressão condicional que pode ser utilizada em qualquer posição onde uma expressão escalar é aceita: `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, `HAVING`.

### 4.1 Forma Pesquisada (Searched CASE)

```sql
CASE
    WHEN condição1 THEN resultado1
    WHEN condição2 THEN resultado2
    ...
    ELSE resultado_padrão
END
```

### 4.2 Classificação de Risco por Score de Fraude

```sql
SELECT
    id_pedido,
    forma_pagamento,
    valor_pagamento,
    avaliacao_fraude.score AS score_fraude,
    CASE
        WHEN avaliacao_fraude.score >= 0.8 THEN 'ALTO'
        WHEN avaliacao_fraude.score >= 0.5 THEN 'MEDIO'
        WHEN avaliacao_fraude.score >= 0.2 THEN 'BAIXO'
        ELSE 'NEGLIGENCIAVEL'
    END AS nivel_risco
FROM db_analytics.pagamentos
ORDER BY score_fraude DESC
LIMIT 30;
```

### 4.3 Categorização de Pedidos por Valor

```sql
SELECT
    ID_PEDIDO,
    PRODUTO,
    VALOR_UNITARIO * QUANTIDADE AS valor_total,
    CASE
        WHEN VALOR_UNITARIO * QUANTIDADE >= 5000 THEN 'PREMIUM'
        WHEN VALOR_UNITARIO * QUANTIDADE >= 1000 THEN 'ALTO'
        WHEN VALOR_UNITARIO * QUANTIDADE >= 200  THEN 'MEDIO'
        ELSE 'BAIXO'
    END AS faixa_valor
FROM db_analytics.pedidos
ORDER BY valor_total DESC
LIMIT 20;
```

---

## 5. Laboratório Prático

**Exercício 1:** Construa um relatório de Pivot com o faturamento total por produto nas UFs SP, RJ, MG e PR. Inclua uma coluna com o total geral.

```sql
SELECT
    PRODUTO,
    SUM(CASE WHEN UF = 'SP' THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END) AS SP,
    SUM(CASE WHEN UF = 'RJ' THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END) AS RJ,
    SUM(CASE WHEN UF = 'MG' THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END) AS MG,
    SUM(CASE WHEN UF = 'PR' THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END) AS PR,
    SUM(VALOR_UNITARIO * QUANTIDADE)                                       AS TOTAL
FROM db_analytics.pedidos
GROUP BY PRODUTO
ORDER BY TOTAL DESC;
```

**Exercício 2:** Classifique cada pagamento em categorias de valor (`< 100`: MICRO, `100-999`: PADRAO, `>= 1000`: ALTO) e calcule o volume total e quantidade de transações por categoria.

```sql
SELECT
    CASE
        WHEN valor_pagamento < 100    THEN 'MICRO'
        WHEN valor_pagamento < 1000   THEN 'PADRAO'
        ELSE 'ALTO'
    END AS categoria_valor,
    COUNT(id_pedido)     AS total_transacoes,
    SUM(valor_pagamento) AS volume_total,
    AVG(valor_pagamento) AS ticket_medio
FROM db_analytics.pagamentos
WHERE status = true
GROUP BY
    CASE
        WHEN valor_pagamento < 100    THEN 'MICRO'
        WHEN valor_pagamento < 1000   THEN 'PADRAO'
        ELSE 'ALTO'
    END
ORDER BY volume_total DESC;
```

---

## 6. Resumo do Módulo

| Operação | Transformação | Implementação no Athena |
|---|---|---|
| Pivot | Linhas → Colunas | `SUM(CASE WHEN col = 'valor' THEN campo ELSE 0 END)` |
| Unpivot | Colunas → Linhas | `UNION ALL` ou `CROSS JOIN (VALUES ...)` |
| Classificação | Valor → Categoria | `CASE WHEN condição THEN resultado END` |

---

[← Módulo 5](../modulo-05/README.md) | [Próximo: Módulo 7 →](../modulo-07/README.md)
