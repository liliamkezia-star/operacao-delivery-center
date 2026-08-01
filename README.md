# Operação do Delivery Center

Dashboard de Business Intelligence para operação de um Delivery Center (marketplace de entregas), cobrindo o período de **01/jan a 30/abr/2021**. Construído em Power BI, com modelo star schema, 57 medidas DAX e 10 páginas de relatório cobrindo visão executiva, financeira, comercial, operação/logística, análise temporal e insights de negócio.

**Autora:** Liliam Kezia Oliveira Souza

---

## ⚠️ Antes de publicar no Git — leia isto

Este repositório contém um arquivo `.pbix`, que é um **binário compactado** (não é texto). Isso tem duas consequências importantes:

1. **Git não consegue mostrar diffs legíveis** de um `.pbix` — cada commit substitui o arquivo inteiro, sem histórico linha a linha de medidas DAX, consultas M ou relacionamentos.
2. **Repositórios acumulam tamanho rapidamente**, já que cada versão salva grava o binário completo de novo (esse arquivo tem ~73 MB).

### Recomendação: publicar também em formato `.pbip` (Power BI Project)

O Power BI Desktop moderno suporta salvar o projeto como **`.pbip`**, que quebra o projeto em arquivos de texto (formato TMDL) — isso sim é git-friendly: diffs legíveis, histórico real de cada medida/coluna/relacionamento, merge/revisão de PR possíveis.

**Como gerar:** no Power BI Desktop, com o arquivo aberto → **Arquivo → Salvar Como** → escolha o tipo **Power BI Project (.pbip)**. Isso cria uma estrutura assim:

```
OperaçãoDoDeliveryCenter.pbip
OperaçãoDoDeliveryCenter.Report/       ← definição do relatório (páginas, visuais) em JSON
OperaçãoDoDeliveryCenter.SemanticModel/ ← modelo (tabelas, DAX, Power Query) em TMDL, texto puro
```

Depois de gerar, é essa pasta (`.Report` + `.SemanticModel` + o `.pbip`) que deve ir pro Git — o `.pbix` pode continuar existindo localmente como cópia de trabalho, mas não precisa (nem deveria, idealmente) ser versionado linha a linha.

Se por algum motivo o `.pbix` precisar mesmo ir para o repositório (ex.: como artefato de distribuição, não como fonte versionada), considere usar **Git LFS** (Large File Storage) pra não inchar o histórico do repo com binários grandes.

---

## Estrutura deste repositório

```
├── README.md                          ← este arquivo
├── CHANGELOG.md                       ← histórico de correções aplicadas
├── .gitignore
├── OPERAÇÃO_DO_DELIVERY_CENTER....pbix ← arquivo de trabalho (Power BI Desktop)
└── docs/
    └── TECHNICAL_DOCUMENTATION.md     ← modelo de dados, DAX, ETL, decisões de design
```

## Como abrir

1. Requer **Power BI Desktop** (Windows).
2. Ao abrir, o parâmetro `pCaminhoFonte` precisa apontar para uma pasta local contendo os 6 arquivos CSV de origem (`orders.csv`, `payments.csv`, `channels.csv`, `hubs.csv`, `stores.csv`, `drivers.csv`). Ajuste em **Transformar Dados → Editar Parâmetros**.
3. Clique em **Atualizar** para carregar os dados.

## Visão geral técnica

- **Modelo:** star schema, 3 tabelas fato (`fPedidos`, `fPagamentos`, `fEntregas`) + 6 dimensões, 8 relacionamentos (todos Many-to-One, filtro em direção única).
- **DAX:** 57 medidas organizadas em 5 pastas temáticas (KPIs, Financeiro, Operacional, Inteligência de Tempo, Auxiliares), mais uma medida de documentação técnica embutida no próprio modelo.
- **Páginas:** Visão Executiva, Financeira, Comercial, Operação, Temporal, Detalhamento (drill-through), Insights, + 3 páginas de tooltip customizado.

Detalhes completos — todas as fórmulas DAX, código Power Query, relacionamentos e decisões de design — estão em [`docs/TECHNICAL_DOCUMENTATION.md`](docs/TECHNICAL_DOCUMENTATION.md).

## Licença / Uso

Defina aqui a licença do projeto (ex.: MIT, ou "uso interno/portfólio pessoal") antes de tornar o repositório público.
