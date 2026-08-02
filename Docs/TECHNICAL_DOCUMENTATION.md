# Documentação Técnica — Operação do Delivery Center

Estado do modelo após as correções registradas em [`CHANGELOG.md`](../CHANGELOG.md). Extraído por conexão direta ao modelo semântico (Power BI Desktop / Analysis Services local).

---

## 1. Modelo de dados (star schema)

### Tabelas

| Tabela | Tipo | Papel |
|---|---|---|
| `fPedidos` | Fato (grão = pedido) | 368.999 linhas — `order_id`, `store_id`, `channel_id`, `Status do Pedido`, `Valor do Pedido`, `Taxa de Entrega`, `Custo de Entrega`, tempos de produção/trânsito/ciclo, `Data do Pedido`, `Hora do Pedido` |
| `fPagamentos` | Fato | `payment_id`, `payment_order_id`, `Valor do Pagamento`, `Taxa de Pagamento`, `Método de Pagamento`, `Status do Pagamento`, `Grupo de Pagamento` |
| `fEntregas` | Fato | `delivery_id`, `delivery_order_id`, `driver_id`, `Distância (km)`, `Status da Entrega` |
| `dCalendario` | Dimensão (calculada, dinâmica) | `Data`, `Ano`, `Mês Núm`, `Mês`, `Mês/Ano`, `Dia`, `Dia da Semana Núm`, `Dia da Semana`, `Semana do Ano`, `Fim de Semana` — gerada via `CALENDAR(MIN/MAX)` sobre `fPedidos[Data do Pedido]` |
| `dLoja` | Dimensão | `store_id`, `hub_id`, `Loja`, `Segmento`, `Preço do Plano`, `Hub`, `Cidade`, `UF`, `Tipo de Plano` (Hub/Cidade/UF mesclados do `dHub` no Power Query) |
| `dCanal` | Dimensão | `channel_id`, `Canal`, `Tipo de Canal` |
| `dEntregador` | Dimensão | `driver_id`, `Modal`, `Tipo de Entregador` — 4.824 linhas |
| `dHub` | Dimensão | `hub_id`, `Hub`, `Cidade`, `UF`, `Latitude`, `Longitude` (mantida separada para o visual de mapa) |
| `dHora` | Dimensão | `Hora`, `Faixa Horária`, `Faixa Ordem` |
| `_Medidas` | Tabela de medidas (desconectada) | 57 medidas DAX em 5 pastas temáticas |

### Relacionamentos (8, todos Many-to-One, filtro em direção única)

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

Todos os relacionamentos são de direção única — um filtro em `dCanal`, por exemplo, afeta `fPedidos`, mas não se propaga de volta para `fPagamentos`/`fEntregas` (que apontam *para* `fPedidos`, não o contrário).

---

## 2. Medidas DAX (57 medidas)

### 📁 00 Documentação

**Documentação Técnica** — medida de texto com o resumo da autora sobre ETL, modelagem e premissas (reproduzida na íntegra ao final deste documento).

### 📁 01 KPIs

```dax
Total de Pedidos = COUNTROWS(fPedidos)

Pedidos Finalizados = CALCULATE([Total de Pedidos], fPedidos[Status do Pedido] = "Finalizado")

Pedidos Cancelados = CALCULATE([Total de Pedidos], fPedidos[Status do Pedido] = "Cancelado")

% Pedidos Cancelados = DIVIDE([Pedidos Cancelados], [Total de Pedidos])

Receita Total = CALCULATE(SUM(fPedidos[Valor do Pedido]), fPedidos[Status do Pedido] = "Finalizado")

Ticket Médio = DIVIDE([Receita Total], [Pedidos Finalizados])

Tempo Médio de Entrega (min) = AVERAGE(fPedidos[Tempo de Ciclo (min)])

% Pedidos no Prazo = DIVIDE([Pedidos no Prazo], [Pedidos com Tempo Medido])

Lojas Ativas = DISTINCTCOUNT(fPedidos[store_id])

Total de Lojas Parceiras = COUNTROWS(dLoja)
```

### 📁 02 Financeiro

```dax
Receita de Pagamentos = CALCULATE(SUM(fPagamentos[Valor do Pagamento]), fPagamentos[Status do Pagamento] = "Pago")

Receita de Taxas (DC) = CALCULATE(SUM(fPagamentos[Taxa de Pagamento]), fPagamentos[Status do Pagamento] = "Pago")

Take Rate = DIVIDE([Receita de Taxas (DC)], [Receita de Pagamentos])

Receita de Frete = CALCULATE(SUM(fPedidos[Taxa de Entrega]), fPedidos[Status do Pedido] = "Finalizado")

Custo Total de Entrega = CALCULATE(SUM(fPedidos[Custo de Entrega]), fPedidos[Status do Pedido] = "Finalizado")

Margem de Entrega = [Receita de Frete] - [Custo Total de Entrega]

Margem de Entrega % = DIVIDE([Margem de Entrega], [Receita de Frete])

Valor em Chargeback = CALCULATE(SUM(fPagamentos[Valor do Pagamento]), fPagamentos[Status do Pagamento] = "Chargeback")

Qtd Chargebacks = CALCULATE(COUNTROWS(fPagamentos), fPagamentos[Status do Pagamento] = "Chargeback")

% Devolução (Proxy) = DIVIDE([Qtd Chargebacks], COUNTROWS(fPagamentos))

Receita por Loja Ativa = DIVIDE([Receita Total], [Lojas Ativas])
```

### 📁 03 Operacional

```dax
Custo Médio de Entrega = CALCULATE(AVERAGE(fPedidos[Custo de Entrega]), fPedidos[Status do Pedido] = "Finalizado")

Tempo Médio de Produção (min) = AVERAGE(fPedidos[Tempo de Produção (min)])

Tempo Médio de Trânsito (min) = AVERAGE(fPedidos[Tempo de Trânsito (min)])

Distância Média (km) = AVERAGE(fEntregas[Distância (km)])

Total de Entregas = COUNTROWS(fEntregas)

Entregas Concluídas = CALCULATE(COUNTROWS(fEntregas), fEntregas[Status da Entrega] = "Entregue")

% Entregas Canceladas = DIVIDE(CALCULATE(COUNTROWS(fEntregas), fEntregas[Status da Entrega] = "Cancelada"), [Total de Entregas])

Entregadores Ativos = DISTINCTCOUNT(fEntregas[driver_id])

Entregas por Entregador = DIVIDE([Entregas Concluídas], [Entregadores Ativos])

% Entregas sem Entregador = DIVIDE(CALCULATE(COUNTROWS(fEntregas), ISBLANK(fEntregas[driver_id])), [Total de Entregas])

% Reentregas = DIVIDE([Total de Entregas], DISTINCTCOUNT(fEntregas[delivery_order_id])) - 1

Média de Pedidos por Dia = DIVIDE([Total de Pedidos], DISTINCTCOUNT(fPedidos[Data do Pedido]))
```

### 📁 04 Inteligência de Tempo

> As 5 medidas de variação mensal abaixo foram corrigidas nesta sessão — ver `CHANGELOG.md` para o histórico do bug e da correção.

```dax
Receita Mês Anterior = CALCULATE([Receita Total], DATEADD(dCalendario[Data], -1, MONTH))

Receita MoM % =
VAR MesRef = IF(HASONEVALUE(dCalendario[Mês Núm]), SELECTEDVALUE(dCalendario[Mês Núm]), MAX(dCalendario[Mês Núm]))
VAR ReceitaMes = CALCULATE([Receita Total], ALL(dCalendario), dCalendario[Mês Núm] = MesRef)
VAR ReceitaAnt = CALCULATE([Receita Total], ALL(dCalendario), dCalendario[Mês Núm] = MesRef - 1)
RETURN DIVIDE(ReceitaMes - ReceitaAnt, ReceitaAnt)

Pedidos Mês Anterior = CALCULATE([Total de Pedidos], DATEADD(dCalendario[Data], -1, MONTH))

Pedidos MoM % =
VAR MesRef = IF(HASONEVALUE(dCalendario[Mês Núm]), SELECTEDVALUE(dCalendario[Mês Núm]), MAX(dCalendario[Mês Núm]))
VAR PedidosMes = CALCULATE([Total de Pedidos], ALL(dCalendario), dCalendario[Mês Núm] = MesRef)
VAR PedidosAnt = CALCULATE([Total de Pedidos], ALL(dCalendario), dCalendario[Mês Núm] = MesRef - 1)
RETURN DIVIDE(PedidosMes - PedidosAnt, PedidosAnt)

Receita Acumulada = CALCULATE([Receita Total], DATESYTD(dCalendario[Data]))

Pedidos Média Móvel 7d = AVERAGEX(DATESINPERIOD(dCalendario[Data], MAX(dCalendario[Data]), -7, DAY), [Total de Pedidos])

Ticket Mês Anterior = CALCULATE([Ticket Médio], DATEADD(dCalendario[Data], -1, MONTH))

Ticket MoM % =
VAR MesRef = IF(HASONEVALUE(dCalendario[Mês Núm]), SELECTEDVALUE(dCalendario[Mês Núm]), MAX(dCalendario[Mês Núm]))
VAR TicketMes = CALCULATE([Ticket Médio], ALL(dCalendario), dCalendario[Mês Núm] = MesRef)
VAR TicketAnt = CALCULATE([Ticket Médio], ALL(dCalendario), dCalendario[Mês Núm] = MesRef - 1)
RETURN DIVIDE(TicketMes - TicketAnt, TicketAnt)

Tempo Médio Mês Anterior = CALCULATE([Tempo Médio de Entrega (min)], DATEADD(dCalendario[Data], -1, MONTH))

Tempo MoM % =
VAR MesRef = IF(HASONEVALUE(dCalendario[Mês Núm]), SELECTEDVALUE(dCalendario[Mês Núm]), MAX(dCalendario[Mês Núm]))
VAR TempoMes = CALCULATE([Tempo Médio de Entrega (min)], ALL(dCalendario), dCalendario[Mês Núm] = MesRef)
VAR TempoAnt = CALCULATE([Tempo Médio de Entrega (min)], ALL(dCalendario), dCalendario[Mês Núm] = MesRef - 1)
RETURN DIVIDE(TempoMes - TempoAnt, TempoAnt)

% No Prazo Mês Anterior = CALCULATE([% Pedidos no Prazo], DATEADD(dCalendario[Data], -1, MONTH))

% No Prazo Var (p.p.) =
VAR MesRef = IF(HASONEVALUE(dCalendario[Mês Núm]), SELECTEDVALUE(dCalendario[Mês Núm]), MAX(dCalendario[Mês Núm]))
VAR PrazoMes = CALCULATE([% Pedidos no Prazo], ALL(dCalendario), dCalendario[Mês Núm] = MesRef)
VAR PrazoAnt = CALCULATE([% Pedidos no Prazo], ALL(dCalendario), dCalendario[Mês Núm] = MesRef - 1)
RETURN IF(NOT ISBLANK(PrazoAnt), (PrazoMes - PrazoAnt) * 100)
```

### 📁 05 Auxiliares

```dax
Premissa SLA (min) = 60

Pedidos com Tempo Medido = CALCULATE(COUNTROWS(fPedidos), NOT(ISBLANK(fPedidos[Tempo de Ciclo (min)])))

Pedidos no Prazo =
VAR sla = [Premissa SLA (min)]
RETURN CALCULATE(
    COUNTROWS(fPedidos),
    FILTER(ALL(fPedidos[Tempo de Ciclo (min)]),
        NOT(ISBLANK(fPedidos[Tempo de Ciclo (min)])) && fPedidos[Tempo de Ciclo (min)] <= sla)
)

Rótulo Período = "Período: " & FORMAT(MIN(dCalendario[Data]), "dd MMM", "pt-BR") & " – " & FORMAT(MAX(dCalendario[Data]), "dd MMM yyyy", "pt-BR")

Título Dinâmico Cidade = "Operação " & IF(ISFILTERED(dLoja[Cidade]), SELECTEDVALUE(dLoja[Cidade], "Múltiplas Cidades"), "Brasil") & " | " & [Rótulo Período]

Receita Total (Card) = "R$ " & FORMAT(DIVIDE([Receita Total], 1000000), "#,0.0") & " Mi"

Entregadores Ativos (Card) = FORMAT([Entregadores Ativos], "#,0")

Dia Mais Forte =
VAR t = TOPN(1, ADDCOLUMNS(VALUES(dCalendario[Dia da Semana]), "@p", [Total de Pedidos]), [@p], DESC)
RETURN MAXX(t, dCalendario[Dia da Semana])

Faixa Mais Forte =
VAR t = TOPN(1, ADDCOLUMNS(VALUES(dHora[Faixa Horária]), "@p", [Total de Pedidos]), [@p], DESC)
VAR completa = MAXX(t, dHora[Faixa Horária])
RETURN TRIM(LEFT(completa, FIND("(", completa & "(") - 1))

% Fim de Semana = DIVIDE(CALCULATE([Total de Pedidos], dCalendario[Fim de Semana] = "Fim de Semana"), [Total de Pedidos])

Média de Pedidos por Dia (Card) = FORMAT([Média de Pedidos por Dia], "#,0")
```

---

## 3. Power Query (ETL) — consultas relevantes

### `dCalendario` (tabela calculada — DAX, não M)

```dax
ADDCOLUMNS(
    CALENDAR(MIN(fPedidos[Data do Pedido]), MAX(fPedidos[Data do Pedido])),
    "Ano", YEAR([Date]),
    "Mês Núm", MONTH([Date]),
    "Mês", FORMAT([Date], "MMM", "pt-BR"),
    "Mês/Ano", FORMAT([Date], "MMM/yy", "pt-BR"),
    "Dia", DAY([Date]),
    "Dia da Semana Núm", WEEKDAY([Date], 2),
    "Dia da Semana", FORMAT([Date], "ddd", "pt-BR"),
    "Semana do Ano", WEEKNUM([Date], 2),
    "Fim de Semana", IF(WEEKDAY([Date], 2) >= 6, "Fim de Semana", "Dia Útil")
)
```

### `dEntregador`

```m
let
    Fonte = Csv.Document(File.Contents(pCaminhoFonte & "drivers.csv"), [Delimiter=",", Columns=3, Encoding=1252, QuoteStyle=QuoteStyle.None]),
    #"Cabeçalhos Promovidos" = Table.PromoteHeaders(Fonte, [PromoteAllScalars=true]),
    #"Espaços Extras Removidos" = Table.TransformColumns(#"Cabeçalhos Promovidos",{{"driver_modal", Text.Trim, type text}, {"driver_type", Text.Trim, type text}}),
    #"Caracteres Invisíveis Removidos" = Table.TransformColumns(#"Espaços Extras Removidos",{{"driver_modal", Text.Clean, type text}, {"driver_type", Text.Clean, type text}}),
    #"Modal: Motoboy Padronizado" = Table.ReplaceValue(#"Caracteres Invisíveis Removidos","MOTOBOY","Motoboy ",Replacer.ReplaceText,{"driver_modal"}),
    #"Modal: Bike Padronizado" = Table.ReplaceValue(#"Modal: Motoboy Padronizado","BIKER","Bike",Replacer.ReplaceText,{"driver_modal"}),
    #"Tipo: Freelance Padronizado" = Table.ReplaceValue(#"Modal: Bike Padronizado","FREELANCE","Freelance",Replacer.ReplaceText,{"driver_type"}),
    #"Tipo: Operador Logístico Padronizado" = Table.ReplaceValue(#"Tipo: Freelance Padronizado","LOGISTIC OPERATOR","Operador Logístico",Replacer.ReplaceText,{"driver_type"}),
    #"Colunas Renomeadas" = Table.RenameColumns(#"Tipo: Operador Logístico Padronizado",{{"driver_type", "Tipo de Entregador"}, {"driver_modal", "Modal"}}),
    #"Tipos Definidos" = Table.TransformColumnTypes(#"Colunas Renomeadas",{{"driver_id", Int64.Type}, {"Modal", type text}})
in
    #"Tipos Definidos"
```

### `fPedidos`

```m
let
    Fonte = Csv.Document(File.Contents(pCaminhoFonte & "orders.csv"), [Delimiter=",", Encoding=1252, QuoteStyle=QuoteStyle.None]),
    #"Cabeçalhos Promovidos" = Table.PromoteHeaders(Fonte, [PromoteAllScalars=true]),
    #"Espaços Extras Removidos" = Table.TransformColumns(#"Cabeçalhos Promovidos", {{"order_status", Text.Trim, type text}}),
    #"Caracteres Invisíveis Removidos" = Table.TransformColumns(#"Espaços Extras Removidos", {{"order_status", Text.Clean, type text}}),
    #"Colunas Redundantes Removidas (chaves, datas soltas, funil)" = Table.RemoveColumns(#"Caracteres Invisíveis Removidos",{"payment_order_id", "delivery_order_id", "order_created_hour", "order_created_minute", "order_created_day", "order_created_month", "order_created_year", "order_moment_accepted", "order_moment_ready", "order_moment_collected", "order_moment_in_expedition", "order_moment_delivering", "order_moment_delivered", "order_moment_finished", "order_metric_collected_time", "order_metric_paused_time", "order_metric_walking_time", "order_metric_expediton_speed_time"}),
    #"Momento Criado Convertido (Localidade EUA)" = Table.TransformColumnTypes(#"Colunas Redundantes Removidas (chaves, datas soltas, funil)", {{"order_moment_created", type datetime}}, "en-US"),
    #"Data do Pedido Extraída" = Table.AddColumn(#"Momento Criado Convertido (Localidade EUA)", "Data", each DateTime.Date([order_moment_created]), type date),
    #"Hora do Pedido Extraída" = Table.AddColumn(#"Data do Pedido Extraída", "Hora", each Time.Hour([order_moment_created]), Int64.Type),
    #"Data e Hora Renomeadas" = Table.RenameColumns(#"Hora do Pedido Extraída",{{"Hora", "Hora do Pedido"}}),
    #"Timestamp Removido (cardinalidade)" = Table.RemoveColumns(#"Data e Hora Renomeadas",{"order_moment_created"}),
    #"Data Renomeada" = Table.RenameColumns(#"Timestamp Removido (cardinalidade)",{{"Data", "Data do Pedido"}}),
    #"Valores e Tempos Convertidos (Localidade EUA)" = Table.TransformColumnTypes(#"Data Renomeada", {{"order_amount", type number}, {"order_delivery_fee", type number}, {"order_delivery_cost", type number}, {"order_metric_production_time", type number}, {"order_metric_transit_time", type number}, {"order_metric_cycle_time", type number}}, "en-US"),
    #"Status: Finalizado Traduzido" = Table.ReplaceValue(#"Valores e Tempos Convertidos (Localidade EUA)","FINISHED","Finalizado",Replacer.ReplaceText,{"order_status"}),
    #"Status: Cancelado Traduzido" = Table.ReplaceValue(#"Status: Finalizado Traduzido","CANCELED","Cancelado",Replacer.ReplaceText,{"order_status"}),
    #"Colunas Traduzidas para PT" = Table.RenameColumns(#"Status: Cancelado Traduzido",{{"order_amount", "Valor do Pedido"}, {"order_delivery_fee", "Taxa de Entrega"}, {"order_delivery_cost", "Custo de Entrega"}, {"order_metric_production_time", "Tempo de Produção (min)"}, {"order_metric_transit_time", "Tempo de Trânsito (min)"}, {"order_metric_cycle_time", "Tempo de Ciclo (min)"}, {"order_status", "Status do Pedido"}}),
    #"Tempos Inválidos Anulados (<0 ou >24h)" = Table.TransformColumns(#"Colunas Traduzidas para PT", {
    {"Tempo de Produção (min)", each if _ <> null and (_ < 0 or _ > 1440) then null else _, type nullable number},
    {"Tempo de Trânsito (min)", each if _ <> null and (_ < 0 or _ > 1440) then null else _, type nullable number},
    {"Tempo de Ciclo (min)",    each if _ <> null and (_ < 0 or _ > 1440) then null else _, type nullable number}
}),
    #"Chaves Tipadas como Inteiro" = Table.TransformColumnTypes(#"Tempos Inválidos Anulados (<0 ou >24h)",{{"store_id", Int64.Type}, {"order_id", Int64.Type}, {"channel_id", Int64.Type}})
in
    #"Chaves Tipadas como Inteiro"
```

> Demais consultas (`dCanal`, `dHub`, `dLoja`, `fPagamentos`, `fEntregas`) seguem o mesmo padrão geral: fonte via `pCaminhoFonte`, limpeza de texto (Trim + Clean), tipagem explícita com localidade `en-US`, tradução de categorias para português. Não reproduzidas aqui por brevidade — extraíveis via `.pbip`/TMDL se necessário.

---

## 4. Documentação técnica da autora (texto integral da medida `Documentação Técnica`)

> **DOCUMENTAÇÃO TÉCNICA — OPERAÇÃO DO DELIVERY CENTER**
>
> **ETL (Power Query):**
> - Fonte parametrizada (`pCaminhoFonte`) — portabilidade e migração cloud
> - Encoding Windows-1252 e tipagem manual com localidade EUA (decimais com ponto e datas americanas) — evita corrupção silenciosa de valores
> - TRIM/CLEAN em textos; categorias traduzidas, nomes próprios preservados
> - `fPedidos` reduzida de 29 para 12 colunas — colunas redundantes e timestamps com 95% de nulos removidos (cardinalidade/memória)
> - Tempos <0 ou >24h e distâncias >50km anulados — registros inválidos não distorcem médias
> - Nulo = 0 apenas quando significa zero real (planos, taxas); mantido nulo quando significa desconhecido (custo de entrega)
> - Mescla `dHub` em `dLoja` — star schema puro
>
> **MODELAGEM:**
> - Star schema: 3 fatos (`fPedidos`, `fPagamentos`, `fEntregas`) + dimensões conformadas
> - Relacionamentos 1:N, filtro em direção única — sem bidirecional, sem N:N
> - `dCalendario` marcada como tabela de data; `dHora` com faixas horárias; chaves técnicas ocultas
>
> **DAX:**
> - 56+ medidas em pastas temáticas; `DIVIDE` em razões; `HASONEVALUE`/`ISBLANK` protegem variações contra contextos ambíguos
> - SLA parametrizado na medida `[Premissa SLA (min)]` = 60 — ajustável em um único ponto
>
> **PREMISSAS:**
> - Receita (GMV) = pedidos finalizados; cancelados excluídos
> - % Devolução via chargeback (proxy — base sem eventos de devolução)
> - Base restrita a jan-abr/2021; sem dados de produtos/itens e de atendimento
>
> *Autora: Liliam Kezia Oliveira Souza · jul/2026*

---

## 5. Estrutura de páginas do relatório

| Página | Visuais | Propósito |
|---|---|---|
| Visão Executiva | 31 | KPIs centrais + variação mensal, visão consolidada |
| Financeira | 30 | Receita, take rate, margem de frete |
| Comercial | 29 | Desempenho por loja/segmento/plano |
| Operação | 39 | Logística, entregadores, mapa de hubs |
| Temporal | 29 | Sazonalidade, heatmap hora × dia da semana |
| Detalhamento | 17 | Drill-through por Loja/Cidade/Canal/Mês |
| Insights | 28 (só texto) | Achados de negócio e premissas, sem gráficos |
| TT Loja / TT Cidade / TT Dia | 5 cada | Tooltips customizados |
