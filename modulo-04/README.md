# Módulo 4: Integração de Tabelas (Joins)

Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## 1. Objetivo do Módulo

Capacitar o participante na integração de múltiplos datasets via operações de junção (`JOIN`) no Amazon Athena. O módulo aborda os tipos fundamentais de join, seu comportamento sobre os datasets do curso e as implicações de performance em consultas distribuídas sobre dados armazenados no S3.

---

## 2. Modelo de Relacionamento dos Datasets

Os três datasets do curso possuem relacionamentos definidos por chaves de negócio:

```
clientes
  └── id  ──────────────────────────────┐
                                         │ ID_CLIENTE (FK)
pedidos                                  │
  ├── ID_PEDIDO  ─────────────────────┐ ◀┘
  └── ID_CLIENTE (FK → clientes.id)   │
                                       │ id_pedido (FK)
pagamentos                             │
  └── id_pedido (FK → pedidos.ID_PEDIDO) ◀┘
```

| Relacionamento | Cardinalidade |
|---|---|
| `clientes` → `pedidos` | 1 cliente : N pedidos |
| `pedidos` → `pagamentos` | 1 pedido : 1 pagamento |

---

## 3. Tipos de JOIN

### 3.1 INNER JOIN

Retorna apenas os registros que possuem correspondência em **ambas** as tabelas.

```sql
-- Pedidos com dados do cliente associado
SELECT
    p.ID_PEDIDO,
    p.PRODUTO,
    p.VALOR_UNITARIO,
    p.QUANTIDADE,
    p.UF,
    c.nome      AS nome_cliente,
    c.email     AS email_cliente
FROM db_analytics.pedidos p
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
LIMIT 20;
```

> Registros de pedidos sem `ID_CLIENTE` correspondente na tabela `clientes` são **excluídos** do resultado.

### 3.2 LEFT JOIN

Retorna **todos** os registros da tabela à esquerda, com os dados correspondentes da tabela à direita quando disponíveis. Quando não há correspondência, os campos da tabela direita aparecem como `NULL`.

```sql
-- Todos os pedidos, com dados do cliente quando disponíveis
SELECT
    p.ID_PEDIDO,
    p.PRODUTO,
    p.UF,
    p.DATA_CRIACAO,
    c.nome  AS nome_cliente,
    c.email AS email_cliente
FROM db_analytics.pedidos p
LEFT JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
LIMIT 20;
```

### 3.3 Identificação de Registros Órfãos

O `LEFT JOIN` combinado com filtro `IS NULL` identifica registros sem correspondência:

```sql
-- Pedidos sem cliente cadastrado
SELECT
    p.ID_PEDIDO,
    p.PRODUTO,
    p.ID_CLIENTE
FROM db_analytics.pedidos p
LEFT JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
WHERE c.id IS NULL;
```

### 3.4 RIGHT JOIN

Retorna **todos** os registros da tabela à direita, independentemente de correspondência à esquerda.

```sql
-- Todos os clientes, com pedidos quando existirem
SELECT
    c.id        AS id_cliente,
    c.nome,
    p.ID_PEDIDO,
    p.PRODUTO
FROM db_analytics.pedidos p
RIGHT JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
LIMIT 20;
```

> Na prática, o `RIGHT JOIN` é intercambiável com o `LEFT JOIN` mediante a inversão da ordem das tabelas. A preferência por `LEFT JOIN` com ordem explícita favorece a legibilidade.

### 3.5 FULL OUTER JOIN

Retorna todos os registros de ambas as tabelas. Quando não há correspondência em um lado, os campos do outro lado aparecem como `NULL`.

```sql
-- Todos os clientes e todos os pedidos, com correspondências quando existirem
SELECT
    c.id        AS id_cliente,
    c.nome,
    p.ID_PEDIDO,
    p.PRODUTO,
    p.UF
FROM db_analytics.clientes c
FULL OUTER JOIN db_analytics.pedidos p
    ON c.id = p.ID_CLIENTE
LIMIT 50;
```

---

## 4. Integração de Três Tabelas

### 4.1 Pedidos com Cliente e Pagamento

```sql
-- Visão consolidada: pedido + cliente + pagamento
SELECT
    p.ID_PEDIDO,
    p.PRODUTO,
    p.VALOR_UNITARIO,
    p.QUANTIDADE,
    p.VALOR_UNITARIO * p.QUANTIDADE AS VALOR_TOTAL,
    p.UF,
    p.DATA_CRIACAO,
    c.nome              AS nome_cliente,
    pg.forma_pagamento,
    pg.valor_pagamento,
    pg.status           AS pagamento_aprovado,
    pg.avaliacao_fraude.score AS score_fraude
FROM db_analytics.pedidos p
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido
LIMIT 50;
```

### 4.2 Pedidos Aprovados por Estado

```sql
SELECT
    p.UF,
    COUNT(p.ID_PEDIDO)                        AS total_pedidos,
    SUM(p.VALOR_UNITARIO * p.QUANTIDADE)      AS faturamento_total,
    COUNT(pg.id_pedido)                       AS pedidos_aprovados
FROM db_analytics.pedidos p
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido
   AND pg.status = true
GROUP BY p.UF
ORDER BY faturamento_total DESC;
```

### 4.3 Clientes com Pedidos Suspeitos de Fraude

```sql
SELECT
    c.id         AS id_cliente,
    c.nome,
    c.email,
    p.ID_PEDIDO,
    p.PRODUTO,
    pg.valor_pagamento,
    pg.avaliacao_fraude.score AS score_fraude
FROM db_analytics.pagamentos pg
INNER JOIN db_analytics.pedidos p
    ON pg.id_pedido = p.ID_PEDIDO
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
WHERE pg.avaliacao_fraude.fraude = true
ORDER BY score_fraude DESC;
```

---

## 5. Considerações de Performance

### 5.1 Broadcast Join vs. Partitioned Join

O Athena/Presto otimiza automaticamente a estratégia de join com base no volume das tabelas:

- **Broadcast Join:** Quando uma das tabelas é suficientemente pequena para ser replicada em todos os workers. Não exige redistribuição de dados entre nós.
- **Partitioned Join (Hash Join):** Quando ambas as tabelas são grandes. O motor distribui os registros por hash da chave de join entre os workers.

Para forçar um broadcast join em tabelas pequenas (quando o otimizador não o faz automaticamente):

```sql
SELECT /*+ BROADCAST(c) */
    p.ID_PEDIDO,
    c.nome
FROM db_analytics.pedidos p
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id;
```

### 5.2 Filtros Antes do Join

Aplicar filtros nas subconsultas antes da operação de join reduz o volume de dados processado:

```sql
-- Filtra pedidos de SP antes de realizar o join com clientes
SELECT
    p.ID_PEDIDO,
    p.PRODUTO,
    c.nome AS nome_cliente
FROM (
    SELECT ID_PEDIDO, PRODUTO, ID_CLIENTE
    FROM db_analytics.pedidos
    WHERE UF = 'SP'
) p
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id;
```

---

## 6. Laboratório Prático

**Exercício 1:** Liste os 10 clientes com maior número de pedidos, exibindo `nome`, `email` e `total_pedidos`.

```sql
SELECT
    c.nome,
    c.email,
    COUNT(p.ID_PEDIDO) AS total_pedidos
FROM db_analytics.clientes c
INNER JOIN db_analytics.pedidos p
    ON c.id = p.ID_CLIENTE
GROUP BY c.nome, c.email
ORDER BY total_pedidos DESC
LIMIT 10;
```

**Exercício 2:** Identifique pedidos que possuem pagamento registrado, mas cujo pagamento foi **reprovado** (`status = false`). Exiba `ID_PEDIDO`, `PRODUTO`, `nome` do cliente e `forma_pagamento`.

```sql
SELECT
    p.ID_PEDIDO,
    p.PRODUTO,
    c.nome AS nome_cliente,
    pg.forma_pagamento,
    pg.valor_pagamento
FROM db_analytics.pedidos p
INNER JOIN db_analytics.pagamentos pg
    ON p.ID_PEDIDO = pg.id_pedido
INNER JOIN db_analytics.clientes c
    ON p.ID_CLIENTE = c.id
WHERE pg.status = false
ORDER BY pg.valor_pagamento DESC;
```

---

## 7. Resumo do Módulo

| Tipo de JOIN | Registros Retornados |
|---|---|
| `INNER JOIN` | Apenas registros com correspondência em ambas as tabelas |
| `LEFT JOIN` | Todos da tabela esquerda + correspondências da direita |
| `RIGHT JOIN` | Todos da tabela direita + correspondências da esquerda |
| `FULL OUTER JOIN` | Todos de ambas as tabelas |

---

[← Módulo 3](../modulo-03/README.md) | [Próximo: Módulo 5 →](../modulo-05/README.md)
