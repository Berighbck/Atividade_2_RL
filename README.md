# CineData Analytics — Atividade 2 · Arquitetura Medalhão (Databricks)

Pipeline ETL **Bronze → Silver → Gold** sobre a base combinada TMDB/IMDb, com Star Schema para BI.

## Notebooks (executar na ordem)

| # | Notebook | O que faz |
|---|----------|-----------|
| 1 | `01_bronze.ipynb` | Cria o schema `bronze`, lê os 5 CSVs do Volume e grava Delta (append) + `ingestion_datetime`; ingere a cotação do dólar (**mock** da API do BCB) |
| 2 | `02_silver.ipynb` | Limpeza, tipagem, tradução, deduplicação e forward fill → 7 tabelas `silver.*` |
| 3 | `03_gold_star_schema.ipynb` | `fact_movies_performance`, 5 dimensões e 3 tabelas-ponte, com checagem de grão e de integridade |
| 4 | `04_gold_analytics.ipynb` | Respostas às 6 perguntas de negócio com `display()` |

## Como rodar
1. No Databricks: **Catalog → seu schema → Create → Volume** (`inputs`) e faça upload dos 5 CSVs da pasta Inputs.
   (O notebook 01 também cria o Volume se ele não existir.)
2. Importe os `.ipynb` no Workspace e ajuste o widget `catalogo` (padrão: `workspace`).
3. Execute 01 → 02 → 03 → 04.

## Mock da API do Banco Central
O endpoint PTAX está instável, então o notebook 01 traz o widget **`usar_mock`** (padrão `true`):
- `true` → `gerar_mock_resposta_bcb()` devolve um JSON **idêntico ao contrato da API real** (`value[].cotacaoCompra` / `dataHoraCotacao`), com 4 boletins por dia útil e nenhum em fins de semana/feriados (semente fixa ⇒ reprodutível).
- `false` → chama a API real; se falhar, usa o mock como fallback.
- A coluna `fonte` em `bronze.tb_cotacao_dolar` registra a origem (`MOCK`, `API_BCB` ou `MOCK_FALLBACK`).

## Principais regras de negócio (todas comentadas no código)
- **Datas:** barra = `dd/MM/yyyy`, hífen = `MM-dd-yyyy`, ISO = `yyyy-MM-dd` (padrões validados por perfil dos dados). Só 2 registros corrompidos viram NULL.
- **Status:** normaliza (`In-Production`, `RELEASED` …) antes de traduzir; inválidos → `Não Informado`.
- **Financeiro:** trata `$`, `USD`, `10.0K`, `34.0M`, `Unknown`; `<= 0` → NULL; lucro só existe com receita **e** orçamento.
- **Métricas:** vírgula decimal corrigida; texto de *column shift* → NULL; notas fora de 0–10 → NULL.
- **Gêneros:** `,` `;` `|` padronizados; só valores do domínio de gêneros TMDB passam.
- **Fato:** 1 linha por filme **lançado**; SKs via `row_number()`.
- **Perguntas 5 e 6:** data-limite = lançamento mais recente com status *Lançado* e data ≤ hoje.

## Observação sobre os nomes dos arquivos
Os CSVs reais são `movies_info_TMDB_IMDB.csv` e `movies_metrics_IMDB_TMDB.csv` (o PDF cita `movies_info_IMDB_TMDB.csv` e `movies_metics…`). O código usa os nomes reais.
