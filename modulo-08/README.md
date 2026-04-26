# Módulo 8: Estatística Aplicada

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Capacitar o participante na aplicação de métricas estatísticas — **desvio padrão** e **variância** — como instrumentos de diagnóstico de instabilidade no faturamento. O módulo demonstra como identificar períodos ou segmentos com comportamento anômalo em séries temporais de receita, utilizando as funções estatísticas nativas do Amazon Athena.

---

## 2. Fundamentos Estatísticos

### 2.1 Variância

A **variância** mede o grau de dispersão dos valores em relação à sua média aritmética. Matematicamente, é a média dos quadrados dos desvios de cada observação em relação à média do conjunto.

**Interpretação:** Uma variância elevada indica que os valores estão amplamente dispersos em torno da média. Uma variância próxima de zero indica que os valores são homogêneos.

**Fórmula (variância da população):**

```
Var(X) = (1/N) * Σ(xᵢ - μ)²
```

Onde:
- `N` = número total de observações
- `xᵢ` = valor da i-ésima observação
- `μ` = média aritmética do conjunto

### 2.2 Desvio Padrão

O **desvio padrão** é a raiz quadrada da variância. Ele retorna a dispersão na **mesma unidade** dos dados originais, o que facilita a interpretação em contextos de negócio.

```
DP(X) = √Var(X)
```

**Interpretação no contexto de faturamento:** Se o faturamento médio mensal é R$ 200.000 e o desvio padrão é R$ 80.000, significa que os valores mensais tipicamente se afastam da média em R$ 80.000 — indicando alta volatilidade operacional.

### 2.3 Coeficiente de Variação (CV)

O **Coeficiente de Variação** normaliza o desvio padrão em relação à média, permitindo comparar a volatilidade entre séries com escalas diferentes:

```
CV = (Desvio Padrão / Média) * 100
```

| CV | Interpretação |
|---|---|
| CV < 15% | Série estável |
| 15% ≤ CV < 30% | Variabilidade moderada |
| CV ≥ 30% | Alta volatilidade — requer investigação |

---

## 3. Funções Estatísticas no Athena

| Função | Descrição |
|---|---|
| `STDDEV(col)` | Desvio padrão amostral (N-1 no denominador) |
| `STDDEV_POP(col)` | Desvio padrão da população (N no denominador) |
| `VARIANCE(col)` | Variância amostral (N-1 no denominador) |
| `VAR_POP(col)` | Variância da população (N no denominador) |
| `AVG(col)` | Média aritmética |
| `CORR(col1, col2)` | Coeficiente de correlação de Pearson |

> **Nota sobre amostra vs. população:** Em análises de negócio sobre dados históricos completos, use `STDDEV_POP` e `VAR_POP`. Use as versões amostrais (`STDDEV`, `VARIANCE`) quando os dados representam uma amostra de uma população maior.

---

## 4. Análise de Volatilidade do Faturamento Mensal

### 4.1 Métricas Globais de Dispersão

```sql
-- Estatísticas de dispersão do faturamento mensal em 2025
WITH faturamento_mensal AS (
    SELECT
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY MONTH(DATE(DATA_CRIACAO))
)
SELECT
    COUNT(mes)                          AS total_meses,
    ROUND(AVG(faturamento), 2)          AS media_mensal,
    ROUND(MIN(faturamento), 2)          AS minimo,
    ROUND(MAX(faturamento), 2)          AS maximo,
    ROUND(MAX(faturamento)
        - MIN(faturamento), 2)          AS amplitude,
    ROUND(STDDEV_POP(faturamento), 2)   AS desvio_padrao,
    ROUND(VAR_POP(faturamento), 2)      AS variancia,
    ROUND(
        100.0 * STDDEV_POP(faturamento)
              / AVG(faturamento),
        2
    )                                   AS coef_variacao_pct
FROM faturamento_mensal;
```

### 4.2 Identificação de Meses Anômalos

Um mês é considerado **anômalo** quando seu faturamento se afasta da média em mais de 1 desvio padrão (critério conservador) ou 2 desvios padrão (critério estrito).

```sql
WITH faturamento_mensal AS (
    SELECT
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY MONTH(DATE(DATA_CRIACAO))
),
estatisticas AS (
    SELECT
        AVG(faturamento)        AS media,
        STDDEV_POP(faturamento) AS desvio_padrao
    FROM faturamento_mensal
)
SELECT
    fm.mes,
    ROUND(fm.faturamento, 2) AS faturamento,
    ROUND(e.media, 2)        AS media_anual,
    ROUND(e.desvio_padrao, 2) AS desvio_padrao,
    ROUND(
        (fm.faturamento - e.media) / e.desvio_padrao,
        2
    ) AS z_score,
    CASE
        WHEN ABS((fm.faturamento - e.media) / e.desvio_padrao) > 2
             THEN 'ANOMALIA SEVERA'
        WHEN ABS((fm.faturamento - e.media) / e.desvio_padrao) > 1
             THEN 'DESVIO SIGNIFICATIVO'
        ELSE 'NORMAL'
    END AS classificacao
FROM faturamento_mensal fm
CROSS JOIN estatisticas e
ORDER BY fm.mes;
```

> **Z-score:** Mede quantos desvios padrão um valor está afastado da média. `|Z| > 2` indica provável anomalia estatística.

---

## 5. Análise de Volatilidade por Segmento

### 5.1 Volatilidade por UF

```sql
-- Ranking de estabilidade de faturamento por estado
WITH faturamento_uf_mes AS (
    SELECT
        UF,
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY UF, MONTH(DATE(DATA_CRIACAO))
)
SELECT
    UF,
    ROUND(AVG(faturamento), 2)        AS media_mensal,
    ROUND(STDDEV_POP(faturamento), 2) AS desvio_padrao,
    ROUND(MIN(faturamento), 2)        AS minimo,
    ROUND(MAX(faturamento), 2)        AS maximo,
    ROUND(
        100.0 * STDDEV_POP(faturamento) / AVG(faturamento),
        2
    )                                  AS coef_variacao_pct,
    CASE
        WHEN 100.0 * STDDEV_POP(faturamento) / AVG(faturamento) < 15
             THEN 'ESTAVEL'
        WHEN 100.0 * STDDEV_POP(faturamento) / AVG(faturamento) < 30
             THEN 'MODERADO'
        ELSE 'VOLATIL'
    END AS diagnostico
FROM faturamento_uf_mes
GROUP BY UF
ORDER BY coef_variacao_pct DESC;
```

### 5.2 Volatilidade por Produto

```sql
WITH faturamento_produto_mes AS (
    SELECT
        PRODUTO,
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY PRODUTO, MONTH(DATE(DATA_CRIACAO))
)
SELECT
    PRODUTO,
    COUNT(mes)                         AS meses_com_venda,
    ROUND(AVG(faturamento), 2)         AS media_mensal,
    ROUND(STDDEV_POP(faturamento), 2)  AS desvio_padrao,
    ROUND(
        100.0 * STDDEV_POP(faturamento) / NULLIF(AVG(faturamento), 0),
        2
    )                                   AS coef_variacao_pct
FROM faturamento_produto_mes
GROUP BY PRODUTO
HAVING COUNT(mes) >= 3
ORDER BY coef_variacao_pct DESC
LIMIT 20;
```

---

## 6. Análise de Volatilidade dos Pagamentos

```sql
-- Volatilidade do valor de pagamentos por forma de pagamento
WITH pagamentos_diarios AS (
    SELECT
        forma_pagamento,
        DATE(data_processamento) AS data,
        SUM(valor_pagamento) AS volume_diario
    FROM db_analytics.pagamentos
    WHERE status = true
      AND YEAR(DATE(data_processamento)) = 2025
    GROUP BY forma_pagamento, DATE(data_processamento)
)
SELECT
    forma_pagamento,
    COUNT(data)                         AS dias_com_transacao,
    ROUND(AVG(volume_diario), 2)        AS media_diaria,
    ROUND(STDDEV_POP(volume_diario), 2) AS desvio_padrao_diario,
    ROUND(MIN(volume_diario), 2)        AS minimo,
    ROUND(MAX(volume_diario), 2)        AS maximo,
    ROUND(
        100.0 * STDDEV_POP(volume_diario) / AVG(volume_diario),
        2
    )                                    AS coef_variacao_pct
FROM pagamentos_diarios
GROUP BY forma_pagamento
ORDER BY coef_variacao_pct DESC;
```

---

## 7. Correlação entre Variáveis

```sql
-- Correlação entre valor do pedido e score de fraude
SELECT
    ROUND(
        CORR(
            p.VALOR_UNITARIO * p.QUANTIDADE,
            pg.avaliacao_fraude.score
        ),
        4
    ) AS correlacao_valor_fraude
FROM db_analytics.pedidos p
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido;
```

> **Interpretação:** Valores de correlação próximos de `+1` indicam correlação positiva forte; próximos de `-1`, correlação negativa forte; próximos de `0`, ausência de correlação linear.

---

## 8. Laboratório Prático

**Exercício 1:** Calcule o coeficiente de variação do faturamento mensal para cada UF em 2025. Identifique as 3 UFs mais instáveis e as 3 mais estáveis.

```sql
WITH uf_mensal AS (
    SELECT
        UF,
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS fat
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY UF, MONTH(DATE(DATA_CRIACAO))
)
SELECT
    UF,
    ROUND(AVG(fat), 2)        AS media,
    ROUND(STDDEV_POP(fat), 2) AS desvio_padrao,
    ROUND(100.0 * STDDEV_POP(fat) / AVG(fat), 2) AS cv_pct
FROM uf_mensal
GROUP BY UF
HAVING COUNT(mes) >= 6
ORDER BY cv_pct DESC;
```

**Exercício 2:** Identifique todos os meses de 2025 em que o faturamento foi superior à média anual em mais de 1 desvio padrão. Exiba o faturamento, z-score e classifique como `ACIMA` ou `ABAIXO` da média.

```sql
WITH mensal AS (
    SELECT
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY MONTH(DATE(DATA_CRIACAO))
),
stats AS (
    SELECT AVG(faturamento) AS media, STDDEV_POP(faturamento) AS dp
    FROM mensal
)
SELECT
    m.mes,
    ROUND(m.faturamento, 2) AS faturamento,
    ROUND(s.media, 2)       AS media,
    ROUND((m.faturamento - s.media) / s.dp, 3) AS z_score,
    CASE WHEN m.faturamento >= s.media THEN 'ACIMA' ELSE 'ABAIXO' END AS posicao
FROM mensal m
CROSS JOIN stats s
WHERE ABS((m.faturamento - s.media) / s.dp) > 1
ORDER BY m.mes;
```

---

## 9. Resumo do Módulo

| Métrica | Função no Athena | Interpretação |
|---|---|---|
| Desvio padrão da população | `STDDEV_POP(col)` | Dispersão em torno da média (mesma unidade) |
| Variância da população | `VAR_POP(col)` | Dispersão ao quadrado |
| Coeficiente de Variação | `STDDEV_POP / AVG * 100` | Volatilidade relativa (%) |
| Z-score | `(valor - média) / dp` | Distância em desvios padrão da média |
| Correlação | `CORR(col1, col2)` | Relação linear entre duas variáveis |

---

[← Módulo 7](../modulo-07/README.md) | [Próximo: Módulo 9 →](../modulo-09/README.md)
