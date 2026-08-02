# Business Case — Delivery Pulse

## Proposta de valor

Um case de Business Intelligence completo — da modelagem de dados brutos à decisão executiva — que transforma 369 mil pedidos de um Delivery Center em um sistema de decisão com 57 métricas de negócio, star schema auditável e uma camada de insights que já aponta, com números, onde a operação está sangrando margem.

## Situação

Um Delivery Center processou quase 369 mil pedidos entre janeiro e abril de 2021, mas operava sem visibilidade estruturada sobre onde estava ganhando e onde estava perdendo dinheiro. Os dados existiam — pedidos, pagamentos, entregas — mas espalhados, sem modelo, sem métricas padronizadas e sem uma leitura executiva do negócio.

## Complicação

Sem um modelo de dados unificado, cada pergunta de negócio (qual loja concentra mais risco? o frete está sendo subsidiado? a operação aguenta os picos de demanda?) exigia análise manual, repetida, sujeita a erro e sem rastreabilidade de como cada número foi calculado.

## Resolução

**Delivery Pulse** resolve isso com um pipeline completo:

- **ETL parametrizado e documentado** (Power Query) — fonte configurável, tratamento de encoding, outliers e nulos com regras de negócio explícitas
- **Modelo star schema com 8 relacionamentos auditados** — 3 fatos, 6 dimensões, direção de filtro única, sem ambiguidade
- **57 medidas DAX organizadas por domínio de negócio** — KPIs, Financeiro, Operacional, Inteligência de Tempo, Auxiliares
- **Relatório de 7 páginas** que separa a leitura executiva (o que aconteceu) da leitura operacional (por que aconteceu, e onde agir), com drill-through dedicado e tooltips customizados

## Impacto — achados de negócio reais

O resultado não é só um dashboard — é uma camada de decisão que já revelou, com números, riscos reais de negócio:

| Achado | Número | Implicação de negócio |
|---|---|---|
| Concentração de receita | **34%** do faturamento vem de 1 única loja | Risco de dependência — perda dessa loja derruba ~1/3 da receita |
| Top 10 lojas | **62%** da receita total | Base de clientes concentrada, pouco diversificada |
| Frete subsidiado | **-R$ 435 mil** de margem negativa em frete | Custo de entrega supera receita de frete — modelo de precificação de frete precisa revisão |
| Concentração horária | **48%** dos pedidos entre 18h–23h | Operação precisa de capacidade elástica para o pico noturno, não distribuição uniforme de recursos |
| Dependência de canal | **90%** dos pedidos via marketplace | Baixa alavancagem de canal próprio — oportunidade direta de margem se migrado |
| Crescimento vs. capacidade | **+54%** em pedidos (melhor mês), com queda de **5 p.p.** na pontualidade no mesmo período | Crescimento não foi acompanhado pela capacidade logística |

São achados que qualquer operador de delivery reconheceria como críveis e acionáveis — não números de exercício.

## Diferenciais técnicos

| Diferencial | Por que é raro em projetos de portfólio |
|---|---|
| Documentação técnica embutida no próprio modelo | Uma medida DAX dedicada documenta ETL, modelagem e premissas — a maioria dos projetos de portfólio não expõe decisões de dado, só o produto final |
| Premissas de negócio explícitas e assumidas | SLA de 60 min declarado como suposição (não dado real), % devolução via chargeback como proxy — transparência que gera confiança técnica |
| Modelo auditado, não só "funcional" | 8 relacionamentos validados, star schema documentado, colunas técnicas ocultadas do usuário final |
| Storytelling executivo real | Achados quantificados — não gráficos soltos sem conclusão |
| Drill-through + tooltips customizados | Nível de interatividade acima do padrão básico de dashboard de portfólio |
| Auditoria técnica completa e pública | Ver [`AUDIT_REPORT.md`](AUDIT_REPORT.md) — avaliação crítica de UX, modelagem, DAX, performance e storytelling, incluindo pontos de melhoria já identificados |
| Histórico de correções documentado | Ver [`../CHANGELOG.md`](../CHANGELOG.md) — evidência de rigor de engenharia, incluindo um bug real de contexto de filtro DAX encontrado e corrigido |

## Premissas de dado assumidas (transparência técnica)

- **Receita (GMV)** considera apenas pedidos com status `Finalizado`; cancelados são excluídos.
- **SLA de entrega** não existe na base original — foi adotado 60 minutos sobre o tempo de ciclo como premissa de negócio, parametrizada em uma única medida (`[Premissa SLA (min)]`) para fácil ajuste.
- **% de devolução** é aproximada via taxa de chargeback, na ausência de um evento de devolução real na base de dados.
- **Escopo temporal** restrito a jan–abr/2021 — sem comparativo ano a ano.
- Sem dados de produtos/itens ou de atendimento ao cliente — análises desse tipo não são calculáveis com a base atual, e isso é declarado explicitamente no relatório em vez de omitido.
