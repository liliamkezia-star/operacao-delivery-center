# Dicionário de Dados — Delivery Pulse

Documentação formal de todas as tabelas e colunas do modelo semântico. Extraído por conexão direta ao modelo (não é aproximação por engenharia reversa de arquivo).

---

## `fPedidos` — Fato (grão: 1 linha por pedido)

368.999 linhas.

| Coluna | Tipo | Oculta | Descrição / Regra de negócio |
|---|---|---|---|
| `order_id` | Inteiro | Não | Chave do pedido. Mantida visível propositalmente — usada na tabela de drill-through da página Detalhamento. |
| `store_id` | Inteiro | Sim | Chave estrangeira para `dLoja`. |
| `channel_id` | Inteiro | Sim | Chave estrangeira para `dCanal`. |
| `Status do Pedido` | Texto | Não | `Finalizado` ou `Cancelado`. Traduzido de `FINISHED`/`CANCELED` no ETL. Receita (GMV) considera somente pedidos `Finalizado`. |
| `Valor do Pedido` | Decimal | Não | Valor bruto do pedido (GMV). |
| `Taxa de Entrega` | Decimal | Não | Valor de frete cobrado do cliente. |
| `Custo de Entrega` | Decimal | Não | Custo pago ao entregador/logística. Pode ser nulo (custo desconhecido — mantido nulo deliberadamente, não zerado). |
| `Tempo de Produção (min)` | Decimal | Não | Tempo entre criação e pedido pronto. Valores <0 ou >1440 min (24h) anulados no ETL como inválidos. |
| `Tempo de Trânsito (min)` | Decimal | Não | Tempo em trânsito até entrega. Mesma regra de invalidação. |
| `Tempo de Ciclo (min)` | Decimal | Não | Tempo total do ciclo do pedido. Base do SLA (ver `[Premissa SLA (min)]` = 60 min). |
| `Data do Pedido` | Data | Não | Extraída do timestamp original via `DateTime.Date()`. Tipo `Date` puro, sem componente de hora (compressão otimizada). Chave de relacionamento com `dCalendario`. |
| `Hora do Pedido` | Inteiro | Não | Hora (0–23) extraída do timestamp original. Chave de relacionamento com `dHora`. |

> Reduzida de 29 para 12 colunas no ETL — colunas redundantes e timestamps intermediários (>95% nulos) removidos por decisão de cardinalidade/memória.

---

## `fPagamentos` — Fato

| Coluna | Tipo | Oculta | Descrição / Regra de negócio |
|---|---|---|---|
| `payment_id` | Inteiro | Sim | Chave do pagamento. |
| `payment_order_id` | Inteiro | Sim | Chave estrangeira para `fPedidos[order_id]`. |
| `Valor do Pagamento` | Decimal | Não | Valor processado. |
| `Taxa de Pagamento` | Decimal | Não | Taxa retida pelo Delivery Center — base da medida `[Receita de Taxas (DC)]`. |
| `Método de Pagamento` | Texto | Não | Forma de pagamento utilizada. |
| `Status do Pagamento` | Texto | Não | `Pago`, `Chargeback`, entre outros. `Chargeback` é usado como proxy de devolução (ver premissa em `BUSINESS_CASE.md`). |
| `Grupo de Pagamento` | Texto | Não | Categoria agregada (7 grupos), calculada via regra de negócio no Power Query. |

---

## `fEntregas` — Fato

| Coluna | Tipo | Oculta | Descrição / Regra de negócio |
|---|---|---|---|
| `delivery_id` | Inteiro | Sim | Chave da entrega. |
| `delivery_order_id` | Inteiro | Sim | Chave estrangeira para `fPedidos[order_id]`. Uma mesma `order_id` pode ter mais de uma entrega associada (reentregas). |
| `driver_id` | Inteiro | Sim | Chave estrangeira para `dEntregador`. Pode ser nulo — base da medida `[% Entregas sem Entregador]`. |
| `Distância (km)` | Decimal | Não | Distância percorrida na entrega. |
| `Status da Entrega` | Texto | Não | `Entregue`, `Cancelada`, entre outros. |

---

## `dCalendario` — Dimensão (tabela calculada, dinâmica)

Gerada via `CALENDAR(MIN(fPedidos[Data do Pedido]), MAX(fPedidos[Data do Pedido]))` — o intervalo se ajusta automaticamente aos dados carregados, sem datas fixas.

| Coluna | Tipo | Descrição |
|---|---|---|
| `Data` | Data | Chave da dimensão. Marcada como coluna de data da tabela (Date Table). |
| `Ano` | Inteiro | `YEAR([Data])`. |
| `Mês Núm` | Inteiro | `MONTH([Data])` — usado para filtros programáticos e nas medidas de variação mensal. |
| `Mês` | Texto | Nome abreviado do mês em pt-BR (`jan`, `fev`...). Coluna usada pelo slicer do relatório. |
| `Mês/Ano` | Texto | Formato `MMM/aa`. |
| `Dia` | Inteiro | Dia do mês. |
| `Dia da Semana Núm` | Inteiro | `WEEKDAY([Data], 2)` — segunda-feira = 1. |
| `Dia da Semana` | Texto | Nome abreviado do dia em pt-BR. |
| `Semana do Ano` | Inteiro | `WEEKNUM([Data], 2)`. |
| `Fim de Semana` | Texto | `"Fim de Semana"` (sáb/dom) ou `"Dia Útil"`. |

---

## `dLoja` — Dimensão

| Coluna | Tipo | Oculta | Descrição / Regra de negócio |
|---|---|---|---|
| `store_id` | Inteiro | Sim | Chave da loja. |
| `hub_id` | Inteiro | Sim | Chave estrangeira para `dHub`. |
| `Loja` | Texto | Não | Nome da loja parceira. |
| `Segmento` | Texto | Não | Categoria de negócio da loja. |
| `Preço do Plano` | Decimal | Não | Mensalidade do plano contratado. Nulo tratado como plano gratuito (regra de negócio, não erro de dado). |
| `Hub` | Texto | Não | Nome do hub — trazido via mesclagem com `dHub` no Power Query, para simplificar consultas de relatório sem precisar navegar até `dHub`. |
| `Cidade` | Texto | Não | Idem — mesclado de `dHub`. |
| `UF` | Texto | Não | Idem — mesclado de `dHub`. |
| `Tipo de Plano` | Texto | Não | `Gratuito` ou `Pago`, derivado de `Preço do Plano` no ETL. |

> `dHub` permanece como tabela separada (relacionada via `hub_id`) porque o visual de mapa da página Operação precisa de `Latitude`/`Longitude`, que não foram trazidas na mesclagem. A duplicidade de `Cidade`/`UF` entre `dLoja` e `dHub` é intencional, não erro de modelagem.

---

## `dCanal` — Dimensão

| Coluna | Tipo | Descrição |
|---|---|---|
| `channel_id` | Inteiro | Chave do canal (oculta). |
| `Canal` | Texto | Nome do canal específico (ex.: nome do marketplace). |
| `Tipo de Canal` | Texto | Categoria agregada — Marketplace vs. Canal Próprio. Base da leitura "90% de dependência de marketplace". |

---

## `dEntregador` — Dimensão

4.824 linhas.

| Coluna | Tipo | Descrição |
|---|---|---|
| `driver_id` | Inteiro | Chave do entregador (oculta). |
| `Modal` | Texto | Modal de entrega (`Motoboy`, `Bike`, entre outros), padronizado no ETL. |
| `Tipo de Entregador` | Texto | `Freelance` ou `Operador Logístico`, padronizado no ETL. |

---

## `dHub` — Dimensão

| Coluna | Tipo | Descrição |
|---|---|---|
| `hub_id` | Inteiro | Chave do hub (oculta). |
| `Hub` | Texto | Nome do hub de distribuição. |
| `Cidade` | Texto | Cidade do hub. |
| `UF` | Texto | Estado do hub. |
| `Latitude` | Decimal | Usada no visual de mapa (página Operação). |
| `Longitude` | Decimal | Usada no visual de mapa (página Operação). |

---

## `dHora` — Dimensão

| Coluna | Tipo | Descrição |
|---|---|---|
| `Hora` | Inteiro | 0–23. Chave de relacionamento com `fPedidos[Hora do Pedido]`. |
| `Faixa Horária` | Texto | Faixa de horário legível (ex.: "18h–23h (Noite)"). |
| `Faixa Ordem` | Inteiro | Coluna auxiliar de ordenação para a `Faixa Horária` não ordenar alfabeticamente. |

---

## `_Medidas` — Tabela de medidas (desconectada)

Não contém dados — existe apenas para organizar as 57 medidas DAX em pastas temáticas. Ver [`TECHNICAL_DOCUMENTATION.md`](TECHNICAL_DOCUMENTATION.md) para todas as fórmulas.

---

## Relacionamentos (referência rápida)

| De (Many) | Coluna | Para (One) | Coluna |
|---|---|---|---|
| `fPedidos` | `channel_id` | `dCanal` | `channel_id` |
| `fEntregas` | `driver_id` | `dEntregador` | `driver_id` |
| `dLoja` | `hub_id` | `dHub` | `hub_id` |
| `fPedidos` | `store_id` | `dLoja` | `store_id` |
| `fPagamentos` | `payment_order_id` | `fPedidos` | `order_id` |
| `fEntregas` | `delivery_order_id` | `fPedidos` | `order_id` |
| `fPedidos` | `Data do Pedido` | `dCalendario` | `Data` |
| `fPedidos` | `Hora do Pedido` | `dHora` | `Hora` |

Todos os relacionamentos são **Many-to-One, filtro em direção única** — sem bidirecionais, sem N:N.
