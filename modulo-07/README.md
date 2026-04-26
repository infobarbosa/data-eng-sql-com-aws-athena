# Módulo 7: Window Functions

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Capacitar o participante na aplicação de **Window Functions** (funções de janela) para análises de ranking, particionamento e séries temporais no Amazon Athena. Ao contrário das funções de agregação com `GROUP BY`, as Window Functions calculam valores sobre um conjunto de registros relacionados sem colapsar o resultado em uma única linha por grupo.

---

## 2. Conceito de Window Function

Uma Window Function opera sobre uma **janela** — um subconjunto definido de registros relacionados ao registro atual. O resultado é calculado para **cada linha individualmente**, mantendo todos os registros no output.

**Sintaxe geral:**

```sql
função() OVER (
    [PARTITION BY coluna_de_partição]
    [ORDER BY coluna_de_ordenação [ASC|DESC]]
    [ROWS|RANGE BETWEEN frame_início AND frame_fim]
)
```

| Cláusula | Função |
|---|---|
| `PARTITION BY` | Divide o dataset em partições independentes para o cálculo |
| `ORDER BY` | Define a sequência dentro de cada partição |
| `ROWS/RANGE BETWEEN` | Define o escopo da janela dentro da partição |

---

## 3. Funções de Ranking

### 3.1 ROW_NUMBER()

Atribui um número sequencial único a cada linha dentro da partição, sem empates.

```sql
-- Número sequencial de pedidos por cliente, ordenado por data
SELECT
    p.ID_CLIENTE,
    c.nome,
    p.ID_PEDIDO,
    p.DATA_CRIACAO,
    p.VALOR_UNITARIO * p.QUANTIDADE AS valor_total,
    ROW_NUMBER() OVER (
        PARTITION BY p.ID_CLIENTE
        ORDER BY DATE(p.DATA_CRIACAO)
    ) AS sequencia_pedido
FROM db_analytics.pedidos p
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
ORDER BY p.ID_CLIENTE, sequencia_pedido;
```

### 3.2 RANK()

Atribui um ranking com empates — registros com valores iguais recebem o mesmo rank, e o próximo rank pula posições.

```sql
-- Ranking de faturamento por UF (com possibilidade de empate)
SELECT
    UF,
    PRODUTO,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento,
    RANK() OVER (
        PARTITION BY UF
        ORDER BY SUM(VALOR_UNITARIO * QUANTIDADE) DESC
    ) AS ranking_produto_na_uf
FROM db_analytics.pedidos
GROUP BY UF, PRODUTO
ORDER BY UF, ranking_produto_na_uf;
```

### 3.3 DENSE_RANK()

Semelhante ao `RANK()`, mas sem pular posições após empates.

```sql
-- Top 3 produtos por UF sem lacunas no ranking
WITH faturamento_produto_uf AS (
    SELECT
        UF,
        PRODUTO,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento,
        DENSE_RANK() OVER (
            PARTITION BY UF
            ORDER BY SUM(VALOR_UNITARIO * QUANTIDADE) DESC
        ) AS ranking
    FROM db_analytics.pedidos
    GROUP BY UF, PRODUTO
)
SELECT *
FROM faturamento_produto_uf
WHERE ranking <= 3
ORDER BY UF, ranking;
```

### 3.4 PERCENT_RANK() e NTILE()

```sql
-- Percentil de faturamento de cada pedido dentro de sua UF
SELECT
    ID_PEDIDO,
    PRODUTO,
    UF,
    VALOR_UNITARIO * QUANTIDADE AS valor_total,
    ROUND(
        PERCENT_RANK() OVER (
            PARTITION BY UF
            ORDER BY VALOR_UNITARIO * QUANTIDADE
        ) * 100,
        2
    ) AS percentil_na_uf
FROM db_analytics.pedidos
ORDER BY UF, percentil_na_uf DESC;
```

```sql
-- Divisão dos pedidos em 4 quartis de valor por UF
SELECT
    ID_PEDIDO,
    UF,
    VALOR_UNITARIO * QUANTIDADE AS valor_total,
    NTILE(4) OVER (
        PARTITION BY UF
        ORDER BY VALOR_UNITARIO * QUANTIDADE
    ) AS quartil
FROM db_analytics.pedidos
ORDER BY UF, quartil;
```

---

## 4. Funções de Valor

### 4.1 LAG() e LEAD()

Permitem acessar o valor de uma linha anterior (`LAG`) ou posterior (`LEAD`) dentro da partição.

```sql
-- Faturamento mensal com variação em relação ao mês anterior
WITH faturamento_mensal AS (
    SELECT
        YEAR(DATE(DATA_CRIACAO))  AS ano,
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento
    FROM db_analytics.pedidos
    GROUP BY
        YEAR(DATE(DATA_CRIACAO)),
        MONTH(DATE(DATA_CRIACAO))
)
SELECT
    ano,
    mes,
    faturamento,
    LAG(faturamento) OVER (ORDER BY ano, mes)                AS faturamento_mes_anterior,
    faturamento - LAG(faturamento) OVER (ORDER BY ano, mes)  AS variacao_absoluta,
    ROUND(
        100.0 * (faturamento - LAG(faturamento) OVER (ORDER BY ano, mes))
              / LAG(faturamento) OVER (ORDER BY ano, mes),
        2
    )                                                         AS variacao_pct
FROM faturamento_mensal
ORDER BY ano, mes;
```

### 4.2 FIRST_VALUE() e LAST_VALUE()

```sql
-- Primeiro e último pedido de cada cliente
SELECT
    p.ID_CLIENTE,
    c.nome,
    p.ID_PEDIDO,
    p.DATA_CRIACAO,
    FIRST_VALUE(p.DATA_CRIACAO) OVER (
        PARTITION BY p.ID_CLIENTE
        ORDER BY DATE(p.DATA_CRIACAO)
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS data_primeiro_pedido,
    LAST_VALUE(p.DATA_CRIACAO) OVER (
        PARTITION BY p.ID_CLIENTE
        ORDER BY DATE(p.DATA_CRIACAO)
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS data_ultimo_pedido
FROM db_analytics.pedidos p
INNER JOIN db_analytics.clientes c ON p.ID_CLIENTE = c.id
ORDER BY p.ID_CLIENTE, DATE(p.DATA_CRIACAO);
```

---

## 5. Funções de Agregação como Window Functions

Qualquer função de agregação pode ser utilizada como Window Function com a cláusula `OVER`. Neste caso, o valor agregado é calculado sobre a janela definida e exibido em cada linha, sem colapsar o resultado.

### 5.1 Participação Percentual no Total

```sql
-- Participação de cada UF no faturamento total
SELECT
    UF,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento_uf,
    SUM(SUM(VALOR_UNITARIO * QUANTIDADE)) OVER () AS faturamento_total,
    ROUND(
        100.0 * SUM(VALOR_UNITARIO * QUANTIDADE)
              / SUM(SUM(VALOR_UNITARIO * QUANTIDADE)) OVER (),
        2
    ) AS participacao_pct
FROM db_analytics.pedidos
GROUP BY UF
ORDER BY faturamento_uf DESC;
```

### 5.2 Acumulado (Running Total)

```sql
-- Faturamento acumulado mês a mês em 2025
WITH mensal AS (
    SELECT
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento_mensal
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY MONTH(DATE(DATA_CRIACAO))
)
SELECT
    mes,
    faturamento_mensal,
    SUM(faturamento_mensal) OVER (
        ORDER BY mes
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS faturamento_acumulado
FROM mensal
ORDER BY mes;
```

### 5.3 Média Móvel de 3 Meses

```sql
WITH mensal AS (
    SELECT
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento_mensal
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY MONTH(DATE(DATA_CRIACAO))
)
SELECT
    mes,
    faturamento_mensal,
    ROUND(
        AVG(faturamento_mensal) OVER (
            ORDER BY mes
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    ) AS media_movel_3m
FROM mensal
ORDER BY mes;
```

---

## 6. Laboratório Prático

**Exercício 1:** Identifique o pedido de maior valor de cada cliente. Exiba `nome`, `ID_PEDIDO`, `PRODUTO`, `valor_total` e a posição desse pedido no ranking de valor do cliente.

```sql
WITH pedidos_rankados AS (
    SELECT
        c.nome,
        p.ID_PEDIDO,
        p.PRODUTO,
        p.VALOR_UNITARIO * p.QUANTIDADE AS valor_total,
        RANK() OVER (
            PARTITION BY p.ID_CLIENTE
            ORDER BY p.VALOR_UNITARIO * p.QUANTIDADE DESC
        ) AS ranking
    FROM db_analytics.pedidos p
    INNER JOIN db_analytics.clientes c ON p.ID_CLIENTE = c.id
)
SELECT nome, ID_PEDIDO, PRODUTO, valor_total
FROM pedidos_rankados
WHERE ranking = 1
ORDER BY valor_total DESC;
```

**Exercício 2:** Calcule a variação percentual mensal do volume de pagamentos aprovados em 2025. Identifique os meses com crescimento e os com retração.

```sql
WITH pagamentos_mensais AS (
    SELECT
        MONTH(DATE(data_processamento)) AS mes,
        SUM(valor_pagamento) AS volume_mensal
    FROM db_analytics.pagamentos
    WHERE YEAR(DATE(data_processamento)) = 2025
      AND status = true
    GROUP BY MONTH(DATE(data_processamento))
)
SELECT
    mes,
    volume_mensal,
    LAG(volume_mensal) OVER (ORDER BY mes) AS volume_anterior,
    ROUND(
        100.0 * (volume_mensal - LAG(volume_mensal) OVER (ORDER BY mes))
              / LAG(volume_mensal) OVER (ORDER BY mes),
        2
    ) AS variacao_pct,
    CASE
        WHEN volume_mensal > LAG(volume_mensal) OVER (ORDER BY mes) THEN 'CRESCIMENTO'
        WHEN volume_mensal < LAG(volume_mensal) OVER (ORDER BY mes) THEN 'RETRAÇÃO'
        ELSE 'ESTAVEL'
    END AS tendencia
FROM pagamentos_mensais
ORDER BY mes;
```

---

## 7. Resumo do Módulo

| Categoria | Funções | Uso Típico |
|---|---|---|
| Ranking | `ROW_NUMBER`, `RANK`, `DENSE_RANK` | Ordenação e seleção de top-N |
| Percentual | `PERCENT_RANK`, `NTILE` | Quartis, percentis, distribuição |
| Deslocamento | `LAG`, `LEAD` | Variação período a período |
| Valor extremo | `FIRST_VALUE`, `LAST_VALUE` | Primeiros e últimos eventos |
| Agregação | `SUM`, `AVG`, `COUNT` + `OVER` | Acumulados, médias móveis, participação |

---

[← Módulo 6](../modulo-06/README.md) | [Próximo: Módulo 8 →](../modulo-08/README.md)
