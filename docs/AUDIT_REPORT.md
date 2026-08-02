# Relatório de Auditoria Técnica — Delivery Pulse

Auditoria própria e crítica do projeto, cobrindo UX/Design, Power Query, Modelagem, DAX, Performance, Storytelling e benchmark de mercado. Publicada de forma transparente — a capacidade de autoavaliar criticamente o próprio trabalho é, em si, um diferencial de maturidade profissional.

---

## 1. Primeira impressão

Identidade visual escura com geometria diagonal em laranja/vermelho — moderna e diferenciada do template padrão do Power BI. Estrutura disciplinada (cabeçalho consistente, cards padronizados, navegação dedicada, drill-through, tooltips customizados, página de premissas explícitas) transmite rigor de consultoria. Pontos de atenção: densidade de 10 números na primeira dobra da Visão Executiva compete por atenção, e a paleta de dados usa o tema padrão do Power BI em vez de uma paleta de marca customizada.

## 2. UX e Design

Grid geral consistente, com pequenas variações de alinhamento (±10px) entre os cards de KPI. Tipografia com hierarquia clara (Segoe UI, com pesos e tamanhos diferenciados por função). Sem ícones ou sparklines nos cards — oportunidade de refinamento visual. Bookmark único ("Limpar Filtros") — funcional, mas sem uso de bookmarks para storytelling condicional.

## 3. Tendências (Fluent Design / Fabric / analytics moderno)

Alinhado com tendências atuais: tema escuro, heatmap hora×dia como visual de destaque, drill-through nativo bem aproveitado. Desalinhado: paleta de dados não customizada, ausência de sparklines e small multiples — recursos cada vez mais padrão em dashboards de referência 2026.

## 4. Auditoria de Power Query (ETL)

Organização acima da média: consultas agrupadas (Fato/Dimensões), nomes de passos autoexplicativos em português. **Achado crítico corrigido:** a consulta `dEntregador` usava um caminho de arquivo absoluto hardcoded em vez do parâmetro `pCaminhoFonte`, quebrando a portabilidade do refresh — corrigido e validado (ver `CHANGELOG.md`). Também identificada e corrigida a ausência de limpeza de texto (`Text.Trim`/`Text.Clean`) em `fPedidos`, que causava duplicação de passos de tradução de status. Pendência remanescente: consolidar o padrão de limpeza de texto (hoje repetido em 6 consultas) em uma função M reutilizável.

## 5. Modelagem

Star schema correto: 3 fatos, 6 dimensões, 8 relacionamentos Many-to-One com filtro em direção única. **Achado crítico corrigido:** a tabela `dCalendario` usava um intervalo de datas fixo (`DATE(2021,1,1)` a `DATE(2021,4,30)`), o que quebraria silenciosamente as medidas de inteligência de tempo assim que novos dados fossem carregados — corrigida para um intervalo dinâmico baseado em `MIN`/`MAX` de `fPedidos[Data do Pedido]`. Colunas técnicas de chave estrangeira devidamente ocultadas do usuário final.

## 6. DAX

57 medidas, nomenclatura consistente, uso disciplinado de `DIVIDE()` (zero divisões cruas), organização em 5 pastas temáticas. **Achado crítico corrigido:** um bug de contexto de filtro nas 5 medidas de variação mensal (MoM) fazia com que retornassem `BLANK` quando um mês era selecionado através do slicer real do relatório — causa raiz identificada como conflito entre filtros de colunas diferentes da mesma tabela de calendário (`Mês` vs. `Mês Núm`), corrigido com `ALL(dCalendario)` dentro do `CALCULATE`. Pendências menores: simplificar a medida `[Pedidos no Prazo]` (usa um padrão `FILTER(ALL())` mais complexo que o necessário) e padronizar a convenção de nomenclatura do símbolo `%` (hoje mistura prefixo e sufixo).

## 7. Performance

Modelo de ~73 MB, porte saudável para o volume de dados. `fPedidos` já foi reduzida de 29 para 12 colunas no ETL — boa prática de compressão já aplicada. Coluna `Data do Pedido` confirmada como tipo `Date` puro (sem componente de hora residual). Ponto de atenção: confirmar se a detecção automática de data/hora do Power BI Desktop está desabilitada, para evitar tabelas de calendário ocultas duplicadas.

## 8. Storytelling

Sequência de páginas segue um arco narrativo coerente: Visão Executiva → Financeira → Comercial → Operação → Temporal → Detalhamento → Insights — do panorama executivo à investigação operacional profunda. A página de Insights, fechando com achados quantificados e premissas explícitas, é o maior ativo de storytelling do projeto.

## 9. Visão de consultoria

Projeto pronto para apresentação a cliente após a correção dos dois achados críticos (portabilidade do ETL e escalabilidade do calendário) — ambos resolvidos nesta auditoria. Documentação de premissas explícita gera confiança técnica pouco comum em entregas de portfólio.

## 10. Visão de portfólio (recrutador de BI)

Demonstra domínio técnico acima de nível júnior (modelagem, DAX avançado, Power Query parametrizado), capacidade analítica real (achados de negócio quantificados, não só execução) e pensamento estruturado (organização de queries e medidas, documentação embutida no modelo). O único deslize de maturidade encontrado (path hardcoded) já foi corrigido e documentado — inclusive esse processo de correção, tornado público, reforça mais do que esconde a maturidade técnica.

## 11. Benchmark de mercado

*Nota de transparência: sem acesso a dashboards internos proprietários de grandes consultorias — comparação baseada em convenções públicas conhecidas do mercado.* Ponto forte real do projeto: documentação de premissas embutida, algo raramente exposto em exemplos públicos de mercado. Ponto de evolução: paletas de dados customizadas e uso de sparklines/microinterações são padrão em portfólios de referência e ainda não implementados aqui.

## 12. Checklist consolidado

| Item | Status | Prioridade |
|---|---|---|
| Path hardcoded em `dEntregador` | ✅ Corrigido | — |
| Calendário fixo → dinâmico | ✅ Corrigido | — |
| Bug de filtro nas 5 medidas MoM | ✅ Corrigido | — |
| Limpeza de texto ausente em `fPedidos` | ✅ Corrigido | — |
| Função M reutilizável (Trim+Clean) | 🟡 Pendente | Média |
| Paleta de dados customizada | 🟡 Pendente | Média |
| Replace global "De"→"de" em `dHub` | 🟡 Pendente | Média |
| Simplificação de `[Pedidos no Prazo]` | 🟡 Pendente | Baixa |
| Padronização de nomenclatura `%` | 🟡 Pendente | Baixa |
| Ícones/sparklines nos cards | 🟡 Pendente | Baixa |

Histórico completo de correções aplicadas: [`../CHANGELOG.md`](../CHANGELOG.md).
