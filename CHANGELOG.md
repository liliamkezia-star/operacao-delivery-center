# Changelog

Todas as correções abaixo foram aplicadas diretamente no modelo semântico (conexão ao vivo via Power BI Desktop), testadas com `EVALUATE` no motor DAX antes de serem salvas.

## [Não lançado] — 2026-08-01

### Corrigido

- **Power Query — `dEntregador`:** consulta usava um caminho de arquivo absoluto, digitado à mão (`C:\Users\pc\Downloads\...`), em vez do parâmetro `pCaminhoFonte` usado por todas as outras consultas. Isso quebrava o refresh assim que o arquivo saísse da máquina/pasta original da autora. Corrigido para `pCaminhoFonte & "drivers.csv"`. *(Portabilidade / Crítico)*

- **Modelagem — `dCalendario`:** tabela calculada usava um intervalo de datas fixo (`CALENDAR(DATE(2021,1,1), DATE(2021,4,30))`). Qualquer dado fora desse intervalo (ex.: meses futuros) ficaria sem correspondência na dimensão de calendário, quebrando silenciosamente as medidas de inteligência de tempo. Corrigido para `CALENDAR(MIN(fPedidos[Data do Pedido]), MAX(fPedidos[Data do Pedido]))` — o intervalo agora se ajusta automaticamente aos dados carregados. *(Escalabilidade / Crítico)*

- **DAX — 5 medidas de variação mensal** (`Receita MoM %`, `Pedidos MoM %`, `Ticket MoM %`, `Tempo MoM %`, `% No Prazo Var (p.p.)`): apresentavam dois problemas relacionados, corrigidos em duas rodadas:
  1. Retornavam `BLANK` (exibido como "--" nos cards) quando nenhum mês estava selecionado no slicer. Corrigido adicionando uma variável `MesRef` que usa o mês selecionado quando existe, ou cai automaticamente no mês mais recente disponível quando não há seleção.
  2. Mesmo após a correção acima, continuavam retornando `BLANK` quando um mês *era* selecionado através do slicer real do relatório (que filtra pela coluna `dCalendario[Mês]`, texto). Causa: o `CALCULATE` interno da medida sobrescrevia apenas o filtro da coluna `dCalendario[Mês Núm]`, mas não removia o filtro pré-existente em `dCalendario[Mês]` — gerando uma interseção de filtros logicamente impossível (ex.: `Mês = "fev"` E `Mês Núm = 1`) para o cálculo do "mês anterior". Corrigido adicionando `ALL(dCalendario)` dentro do `CALCULATE`, limpando todos os filtros da tabela antes de aplicar o filtro específico de mês de referência.

- **Power Query — `fPedidos`:** faltava limpeza de texto (`Text.Trim`/`Text.Clean`) na coluna `order_status`, ao contrário das demais 5 consultas do modelo. Isso causava duas variantes do mesmo valor na fonte (`"CANCELED "` com espaço e `"CANCELED"` sem espaço), exigindo dois passos separados de tradução para "Cancelado". Corrigido: adicionada limpeza logo após a promoção de cabeçalhos (mesmo padrão das outras consultas) e consolidados os dois passos de tradução em um só.

### Restaurado

- **Agrupamento de consultas** (pastas "Dimensões" / "Fato" no painel de Consultas): foi perdido durante um salvamento do arquivo por motivo não identificado com certeza (não foi causado diretamente pelas edições de conteúdo acima — todas as 7 consultas de importação perderam o agrupamento no mesmo instante). Restaurado manualmente para as pastas originais.

### Verificado (sem alterações necessárias)

- Tipo da coluna `fPedidos[Data do Pedido]`: confirmado como `Date` puro, sem componente de hora residual — boa prática de compressão já estava correta.
- Colunas técnicas de chave estrangeira (`channel_id`, `driver_id`, `payment_order_id`, `delivery_order_id`, `payment_id`, `delivery_id`, `store_id`, `hub_id`): todas confirmadas como ocultas (`isHidden: true`) — não expostas desnecessariamente na lista de campos do relatório.
- Relação `dLoja` ↔ `dHub`: aparente duplicidade de colunas (`Cidade`/`UF`/`Hub`) esclarecida — é uma mesclagem intencional feita no Power Query para simplificar o relatório, com `dHub` mantida separada apenas porque o visual de mapa precisa das colunas `Latitude`/`Longitude`, que não foram trazidas na mesclagem.

## Pendências conhecidas (não corrigidas nesta sessão)

- Função M reutilizável para consolidar o padrão de limpeza de texto (Trim + Clean), hoje repetido em 6 consultas.
- Correção do `Table.ReplaceValue` global ("De" → "de") em `dHub[Cidade]`, hoje um replace amplo demais.
- Paleta de dados customizada (o modelo usa o tema de cores padrão do Power BI, não uma paleta de marca).
- Simplificação da medida `[Pedidos no Prazo]` (usa um padrão `FILTER(ALL())` mais complexo que o necessário).
- Padronização da convenção de nomenclatura do símbolo `%` nas medidas (prefixo vs. sufixo, hoje inconsistente).
