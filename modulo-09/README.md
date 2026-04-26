# Módulo 9: Projeto Final

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Conduzir uma **investigação analítica completa** sobre a volatilidade do faturamento em 2025, integrando todas as competências desenvolvidas nos módulos anteriores: DDL, joins, agregações, reestruturação, window functions e estatística aplicada. O projeto simula uma demanda real de análise de negócio com entregáveis formais.

---

## 2. Contexto do Problema de Negócio

A diretoria financeira identificou **instabilidade recorrente no faturamento mensal** ao longo de 2025 e solicitou uma investigação analítica com as seguintes perguntas de negócio:

| # | Pergunta de Negócio |
|---|---|
| 1 | Qual foi a evolução mensal do faturamento em 2025 e quais meses apresentaram desvios significativos? |
| 2 | Quais estados e produtos contribuem para a volatilidade observada? |
| 3 | Existe concentração de risco em alguma forma de pagamento ou segmento de clientes? |
| 4 | Qual é o perfil dos pedidos de alto valor e qual sua taxa de aprovação? |
| 5 | Existe correlação entre o valor do pedido e o score de risco de fraude? |

---

## 3. Fase 1: Configuração do Ambiente

Verifique se as três tabelas estão registradas e operacionais:

```sql
-- Validação das tabelas do catálogo
SHOW TABLES IN db_analytics;

-- Contagem de registros por tabela
SELECT 'clientes'   AS tabela, COUNT(*) AS total FROM db_analytics.clientes
UNION ALL
SELECT 'pedidos',   COUNT(*) FROM db_analytics.pedidos
UNION ALL
SELECT 'pagamentos', COUNT(*) FROM db_analytics.pagamentos;
```

---

## 4. Fase 2: Análise da Evolução Mensal do Faturamento

### 4.1 Série Temporal com Métricas de Dispersão

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
    SELECT
        AVG(faturamento)        AS media,
        STDDEV_POP(faturamento) AS dp
    FROM mensal
)
SELECT
    m.mes,
    ROUND(m.faturamento, 2)  AS faturamento,
    ROUND(s.media, 2)         AS media_anual,
    ROUND(s.dp, 2)            AS desvio_padrao,
    ROUND((m.faturamento - s.media) / s.dp, 3) AS z_score,
    m.faturamento - LAG(m.faturamento) OVER (ORDER BY m.mes) AS var_absoluta_mes_anterior,
    ROUND(
        100.0 * (m.faturamento - LAG(m.faturamento) OVER (ORDER BY m.mes))
              / LAG(m.faturamento) OVER (ORDER BY m.mes),
        2
    ) AS var_pct_mes_anterior,
    SUM(m.faturamento) OVER (ORDER BY m.mes ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        AS faturamento_acumulado,
    CASE
        WHEN ABS((m.faturamento - s.media) / s.dp) > 2 THEN 'ANOMALIA SEVERA'
        WHEN ABS((m.faturamento - s.media) / s.dp) > 1 THEN 'DESVIO SIGNIFICATIVO'
        ELSE 'NORMAL'
    END AS classificacao
FROM mensal m
CROSS JOIN stats s
ORDER BY m.mes;
```

---

## 5. Fase 3: Análise por Estado (UF)

### 5.1 Volatilidade do Faturamento por UF

```sql
WITH uf_mes AS (
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
    COUNT(mes)                                AS meses_ativos,
    ROUND(AVG(faturamento), 2)               AS media_mensal,
    ROUND(STDDEV_POP(faturamento), 2)        AS desvio_padrao,
    ROUND(MIN(faturamento), 2)               AS minimo_mensal,
    ROUND(MAX(faturamento), 2)               AS maximo_mensal,
    ROUND(100.0 * STDDEV_POP(faturamento)
               / AVG(faturamento), 2)         AS coef_variacao_pct,
    RANK() OVER (
        ORDER BY 100.0 * STDDEV_POP(faturamento) / AVG(faturamento) DESC
    ) AS ranking_volatilidade
FROM uf_mes
GROUP BY UF
HAVING COUNT(mes) >= 6
ORDER BY coef_variacao_pct DESC;
```

### 5.2 Contribuição de Cada UF para o Total Anual

```sql
SELECT
    UF,
    SUM(VALOR_UNITARIO * QUANTIDADE) AS faturamento_uf,
    ROUND(
        100.0 * SUM(VALOR_UNITARIO * QUANTIDADE)
              / SUM(SUM(VALOR_UNITARIO * QUANTIDADE)) OVER (),
        2
    ) AS participacao_pct,
    RANK() OVER (
        ORDER BY SUM(VALOR_UNITARIO * QUANTIDADE) DESC
    ) AS ranking_faturamento
FROM db_analytics.pedidos
WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
GROUP BY UF
ORDER BY faturamento_uf DESC;
```

---

## 6. Fase 4: Análise por Produto

### 6.1 Top 10 Produtos e sua Volatilidade

```sql
WITH produto_mes AS (
    SELECT
        PRODUTO,
        MONTH(DATE(DATA_CRIACAO)) AS mes,
        SUM(VALOR_UNITARIO * QUANTIDADE) AS receita
    FROM db_analytics.pedidos
    WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
    GROUP BY PRODUTO, MONTH(DATE(DATA_CRIACAO))
),
produto_stats AS (
    SELECT
        PRODUTO,
        ROUND(SUM(receita), 2)             AS receita_anual,
        ROUND(AVG(receita), 2)             AS media_mensal,
        ROUND(STDDEV_POP(receita), 2)      AS desvio_padrao,
        ROUND(100.0 * STDDEV_POP(receita)
                    / AVG(receita), 2)      AS cv_pct,
        COUNT(mes)                         AS meses_com_venda
    FROM produto_mes
    GROUP BY PRODUTO
    HAVING COUNT(mes) >= 3
)
SELECT
    PRODUTO,
    receita_anual,
    media_mensal,
    desvio_padrao,
    cv_pct,
    meses_com_venda,
    RANK() OVER (ORDER BY receita_anual DESC) AS ranking_receita
FROM produto_stats
ORDER BY receita_anual DESC
LIMIT 10;
```

---

## 7. Fase 5: Análise de Pagamentos e Risco

### 7.1 Consolidação por Forma de Pagamento

```sql
SELECT
    pg.forma_pagamento,
    COUNT(pg.id_pedido)                             AS total_transacoes,
    SUM(CASE WHEN pg.status = true  THEN 1 ELSE 0 END) AS aprovadas,
    SUM(CASE WHEN pg.status = false THEN 1 ELSE 0 END) AS reprovadas,
    ROUND(100.0 * SUM(CASE WHEN pg.status = true THEN 1 ELSE 0 END)
               / COUNT(pg.id_pedido), 2)              AS taxa_aprovacao_pct,
    ROUND(SUM(CASE WHEN pg.status = true
              THEN pg.valor_pagamento ELSE 0 END), 2) AS volume_aprovado,
    ROUND(AVG(pg.avaliacao_fraude.score), 4)          AS score_fraude_medio,
    ROUND(STDDEV_POP(pg.avaliacao_fraude.score), 4)   AS dp_score_fraude
FROM db_analytics.pagamentos pg
GROUP BY pg.forma_pagamento
ORDER BY volume_aprovado DESC;
```

### 7.2 Pedidos de Alto Risco com Dados Completos

```sql
SELECT
    p.ID_PEDIDO,
    p.PRODUTO,
    p.UF,
    p.DATA_CRIACAO,
    ROUND(p.VALOR_UNITARIO * p.QUANTIDADE, 2) AS valor_total,
    c.nome                                     AS nome_cliente,
    c.email                                    AS email_cliente,
    pg.forma_pagamento,
    pg.valor_pagamento,
    pg.status                                  AS pagamento_aprovado,
    ROUND(pg.avaliacao_fraude.score, 4)        AS score_fraude,
    CASE
        WHEN pg.avaliacao_fraude.score >= 0.8 THEN 'CRITICO'
        WHEN pg.avaliacao_fraude.score >= 0.6 THEN 'ALTO'
        WHEN pg.avaliacao_fraude.score >= 0.4 THEN 'MEDIO'
        ELSE 'BAIXO'
    END AS nivel_risco
FROM db_analytics.pedidos p
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido
WHERE pg.avaliacao_fraude.score >= 0.6
ORDER BY score_fraude DESC, valor_total DESC
LIMIT 50;
```

### 7.3 Correlação Valor do Pedido x Score de Fraude

```sql
SELECT
    ROUND(
        CORR(
            p.VALOR_UNITARIO * p.QUANTIDADE,
            pg.avaliacao_fraude.score
        ),
        4
    ) AS correlacao_valor_fraude,
    COUNT(*) AS total_registros
FROM db_analytics.pedidos p
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido
WHERE YEAR(DATE(p.DATA_CRIACAO)) = 2025;
```

---

## 8. Fase 6: Pivot de Faturamento Mensal por UF

```sql
-- Relatório matricial: UF (linhas) x Mês (colunas)
SELECT
    UF,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 1  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS jan,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 2  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS fev,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 3  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS mar,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 4  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS abr,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 5  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS mai,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 6  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS jun,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 7  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS jul,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 8  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS ago,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 9  THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS set,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 10 THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS out,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 11 THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS nov,
    ROUND(SUM(CASE WHEN MONTH(DATE(DATA_CRIACAO)) = 12 THEN VALOR_UNITARIO * QUANTIDADE ELSE 0 END), 2) AS dez,
    ROUND(SUM(VALOR_UNITARIO * QUANTIDADE), 2)                                                           AS total_anual
FROM db_analytics.pedidos
WHERE YEAR(DATE(DATA_CRIACAO)) = 2025
GROUP BY UF
ORDER BY total_anual DESC;
```

---

## 9. Fase 7: Análise de Clientes Estratégicos

### 9.1 Top 10 Clientes por Faturamento com Indicadores de Risco

```sql
WITH cliente_stats AS (
    SELECT
        c.id,
        c.nome,
        c.email,
        COUNT(DISTINCT p.ID_PEDIDO)              AS total_pedidos,
        SUM(p.VALOR_UNITARIO * p.QUANTIDADE)     AS faturamento_total,
        AVG(p.VALOR_UNITARIO * p.QUANTIDADE)     AS ticket_medio,
        AVG(pg.avaliacao_fraude.score)           AS score_fraude_medio,
        SUM(CASE WHEN pg.status = false THEN 1 ELSE 0 END) AS pagamentos_reprovados
    FROM db_analytics.clientes c
    INNER JOIN db_analytics.pedidos p
        ON c.id = p.ID_CLIENTE
    INNER JOIN db_analytics.pagamentos pg
        ON p.ID_PEDIDO = pg.id_pedido
    WHERE YEAR(DATE(p.DATA_CRIACAO)) = 2025
    GROUP BY c.id, c.nome, c.email
)
SELECT
    nome,
    email,
    total_pedidos,
    ROUND(faturamento_total, 2) AS faturamento_total,
    ROUND(ticket_medio, 2)      AS ticket_medio,
    ROUND(score_fraude_medio, 4) AS score_fraude_medio,
    pagamentos_reprovados,
    RANK() OVER (ORDER BY faturamento_total DESC) AS ranking_faturamento
FROM cliente_stats
ORDER BY faturamento_total DESC
LIMIT 10;
```

---

## 10. Entregáveis do Projeto

Ao concluir todas as fases, o participante deve ser capaz de responder formalmente às perguntas de negócio apresentadas na Seção 2:

| Pergunta | Query de Referência |
|---|---|
| 1. Evolução mensal e anomalias | Seção 4.1 |
| 2. Estados e produtos com volatilidade | Seções 5.1 e 6.1 |
| 3. Concentração de risco por pagamento | Seção 7.1 |
| 4. Perfil dos pedidos de alto valor | Seção 7.2 |
| 5. Correlação valor x fraude | Seção 7.3 |

### Critérios de Avaliação

| Critério | Peso |
|---|---|
| Corretude das queries (sintaxe e semântica) | 30% |
| Pertinência analítica das consultas em relação às perguntas | 25% |
| Uso adequado de CTEs para organização lógica | 20% |
| Aplicação correta de métricas estatísticas | 15% |
| Clareza dos aliases e legibilidade do código | 10% |

---

## 11. Resumo do Projeto

Este projeto integra:

- **Módulo 2:** Criação e validação das tabelas externas.
- **Módulo 3:** Filtros temporais (`WHERE YEAR(...) = 2025`).
- **Módulo 4:** Integração de três tabelas via `INNER JOIN`.
- **Módulo 5:** Agregações mensais e por segmento com `GROUP BY`.
- **Módulo 6:** Relatório matricial com Pivot via `CASE WHEN`.
- **Módulo 7:** `LAG`, `RANK`, `SUM() OVER` para séries temporais.
- **Módulo 8:** `STDDEV_POP`, `VAR_POP`, z-score e coeficiente de variação.

---

[← Módulo 8](../modulo-08/README.md) | [← Início](../README.md)
