# Case Study — Análise de Vendas, Pagamentos e Clientes-Chave

**Simulação de um pedido real de negócio, do problema à recomendação**

## Contexto

Este projeto simula uma situação comum no dia a dia de um(a) analista de dados: um pedido de análise recebido em linguagem de negócio, sem especificação técnica.

> *"Ando com a sensação de que estamos a perder dinheiro com pagamentos que não se concretizam, e não sei se todas as categorias de produto estão a ter o mesmo desempenho. Também quero perceber melhor quem são os nossos clientes mais valiosos, para pensarmos numa ação de fidelização. Preciso de 3 a 4 coisas concretas que me ajudem a decidir onde focar."*
> — Diretora Comercial (cenário simulado)

O objetivo não era só escrever SQL — era o processo completo: traduzir um pedido ambíguo em perguntas respondíveis, extrair os dados, e comunicar os resultados de forma acionável para quem não lê código.

**Stack utilizada:** SQL (SQLite), com CTEs, subqueries, `CASE WHEN` e agregações.

**Metodologia:** exercício de prática autodirigida, desenvolvido com apoio do Claude (Anthropic) como parceiro de estudo — a IA simulou o pedido de negócio e testou tecnicamente cada query, mas as perguntas de análise, as decisões metodológicas (definição de prazos, separação de perguntas ambíguas) e as correções ao longo do processo foram minhas. Uso ferramentas de IA de forma ativa no meu processo de aprendizagem e no dia a dia de trabalho, como forma de acelerar a curva de aprendizagem e validar raciocínio técnico.

---

## 1. Traduzir o pedido em perguntas de dados

Antes de qualquer query, o pedido foi decomposto em perguntas concretas e respondíveis:

1. Qual o valor total de pagamentos cancelados e/ou estornados, e qual a sua percentagem em relação aos aprovados?
2. Quais os 10 clientes que mais compraram num prazo de 6 meses?
3. Em quais categorias os clientes do top 10 mais compraram, e qual o total vendido?
4. Qual o volume de vendas total, num prazo de 12 meses, com status aprovado, por categoria?

Duas decisões metodológicas relevantes, tomadas conscientemente:

- **Definição de prazo:** um prazo de análise não é automático — depende do que a pergunta de negócio precisa. Para identificar clientes valiosos (pergunta 2), um prazo curto demais (ex.: 1 mês) excluiria clientes fiéis que compram com menor frequência mas maior valor; por isso optou-se por uma janela de 6 meses.
- **Separação de perguntas fundidas:** a formulação inicial tentava responder a três coisas em uma só pergunta (quem compra mais + em que período + em que categorias). Separar em perguntas distintas resultou em queries mais simples e em resultados mais fáceis de validar.

---

## 2. Análise técnica (SQL)

### Pergunta 1 — Pagamentos recusados/estornados vs. aprovados

```sql
SELECT
    SUM(CASE WHEN status IN ('recusado', 'estornado') THEN valor ELSE 0 END) AS total_recusados_estornados,
    SUM(CASE WHEN status = 'aprovado' THEN valor ELSE 0 END) AS total_aprovados,
    ROUND(
        (SUM(CASE WHEN status IN ('recusado', 'estornado') THEN valor ELSE 0 END) * 100.0) /
        NULLIF(SUM(CASE WHEN status = 'aprovado' THEN valor ELSE 0 END), 0), 2
    ) AS percentual_sobre_aprovados
FROM pagamentos;
```

### Pergunta 2 — Top 10 clientes (últimos 6 meses)

```sql
WITH ultimos_6_meses AS (
    SELECT DISTINCT strftime('%Y-%m', data_pedido) AS mes
    FROM pedidos
    ORDER BY mes DESC
    LIMIT 6
)
SELECT clientes.nome, SUM(pagamentos.valor) AS valor_total_gasto
FROM pedidos
JOIN pagamentos ON pedidos.pedido_id = pagamentos.pedido_id
JOIN clientes ON pedidos.cliente_id = clientes.cliente_id
WHERE pagamentos.status = 'aprovado'
  AND strftime('%Y-%m', pedidos.data_pedido) IN (SELECT mes FROM ultimos_6_meses)
GROUP BY clientes.cliente_id, clientes.nome
ORDER BY valor_total_gasto DESC
LIMIT 10;
```

### Pergunta 3 — Categorias preferidas do top 10

```sql
WITH ultimos_6_meses AS (
    SELECT DISTINCT strftime('%Y-%m', data_pedido) AS mes
    FROM pedidos ORDER BY mes DESC LIMIT 6
),
top_10_clientes AS (
    SELECT pedidos.cliente_id
    FROM pedidos
    JOIN pagamentos ON pedidos.pedido_id = pagamentos.pedido_id
    WHERE pagamentos.status = 'aprovado'
      AND strftime('%Y-%m', pedidos.data_pedido) IN (SELECT mes FROM ultimos_6_meses)
    GROUP BY pedidos.cliente_id
    ORDER BY SUM(pagamentos.valor) DESC
    LIMIT 10
)
SELECT categorias.nome AS categoria,
       SUM(pedidos.quantidade) AS total_itens_vendidos,
       SUM(pagamentos.valor) AS valor_total_vendas
FROM pedidos
JOIN pagamentos ON pedidos.pedido_id = pagamentos.pedido_id
JOIN produtos ON pedidos.produto_id = produtos.produto_id
JOIN categorias ON produtos.categoria_id = categorias.categoria_id
WHERE pagamentos.status = 'aprovado'
  AND pedidos.cliente_id IN (SELECT cliente_id FROM top_10_clientes)
  AND strftime('%Y-%m', pedidos.data_pedido) IN (SELECT mes FROM ultimos_6_meses)
GROUP BY categorias.categoria_id, categorias.nome
ORDER BY valor_total_vendas DESC;
```

### Pergunta 4 — Volume de vendas por categoria (últimos 12 meses)

```sql
WITH ultimos_meses AS (
    SELECT DISTINCT strftime('%Y-%m', data_pedido) AS mes
    FROM pedidos ORDER BY mes DESC LIMIT 12
)
SELECT categorias.nome AS categoria, SUM(pagamentos.valor) AS total_vendas
FROM produtos
JOIN pedidos ON produtos.produto_id = pedidos.produto_id
JOIN pagamentos ON pagamentos.pedido_id = pedidos.pedido_id
JOIN categorias ON categorias.categoria_id = produtos.categoria_id
WHERE pagamentos.status = 'aprovado'
  AND strftime('%Y-%m', pedidos.data_pedido) IN (SELECT mes FROM ultimos_meses)
GROUP BY produtos.categoria_id
ORDER BY total_vendas DESC;
```

---

## 3. Resultados, traduzidos para o negócio

**Pagamentos recusados/estornados**
O percentual de recusados e estornados é de cerca de 30,65% (R$4.556,70) sobre o valor de aprovados. Sugere-se validar com a equipa comercial as causas mais frequentes (limite de crédito, alternativas de produto com menor valor, prazos de pagamento, crédito em loja) antes de decidir uma ação corretiva.

**Clientes mais valiosos (últimos 6 meses)**
No top 10, a cliente com maior valor total foi Marina Castro (R$1.429,50); o décimo lugar foi João Pereira (R$299,70).

**Categorias preferidas do top 10**
Entre os clientes mais valiosos, a categoria Eletrónicos lidera com R$3.088,80, seguida de Moda (R$969,30), Casa e Decoração (R$739,20), Esporte (R$539,50) e Beleza (R$299,70).

**Volume de vendas por categoria (12 meses)**
As categorias não têm volume balanceado: Eletrónicos ultrapassa R$5.000, Casa e Decoração aproxima-se de R$4.000, Moda ronda R$3.000, Esporte quase R$2.000, e Beleza fica abaixo de R$1.000.

*Nota pessoal, não derivada diretamente da análise: por experiência de mercado, Moda e Beleza costumam ser categorias complementares (público semelhante, compra por impulso associada) — pode valer a pena testar uma campanha conjunta entre as duas, embora isto não tenha sido confirmado estatisticamente nesta análise.*

---

## 4. Proposta de acompanhamento contínuo (Power BI)

Para transformar esta análise pontual num painel de acompanhamento mensal:

- **Pagamentos:** gráfico de linha com a evolução mensal da percentagem aprovado/recusado/estornado — permite ver tendência, não só o valor do momento.
- **Clientes-chave:** tabela de ranking com posição, nome, categoria preferida, contacto e valor total — pensada para ação direta (campanha de fidelização), não só visualização.
- **Vendas por categoria:** gráfico de colunas empilhadas, com os **meses no eixo X** e as categorias empilhadas dentro de cada barra — permite ver, na mesma vista, se o total mensal está a subir/descer e como a composição por categoria muda ao longo do tempo.

---

## Competências demonstradas

`SQL (CTEs, subqueries, CASE WHEN, agregações)` · `Tradução de requisitos de negócio em perguntas analíticas` · `Comunicação de dados para audiência não-técnica` · `Pensamento crítico sobre correlação vs. opinião` · `Design de dashboard orientado a decisão`
