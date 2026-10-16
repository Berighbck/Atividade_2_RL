# CineData Analytics — Atividade RocketLab 2026.2 · Pipeline Medalhão + Workflow (Databricks)

Pipeline ETL **Landing → Bronze → Silver → Gold** (Star Schema) sobre a base TMDB/IMDb, orquestrado por um **Databricks Workflow** com 3 tasks encadeadas.

## Arquivos do repositório

| Arquivo | O que é |
|---|---|
| `Landing_to_Bronze.ipynb` | Task `to_Bronze`: CSVs do Volume + cotação do dólar (mock/API do BCB) → `bronze.*` (Delta, append, `ingestion_datetime`) |
| `Bronze_to_Silver.ipynb` | Task `to_Silver`: limpeza, tipagem, tradução, deduplicação e forward fill → 7 tabelas `silver.*` |
| `Silver_to_Gold.ipynb` | Task `to_Gold`: Star Schema (fato, 5 dimensões, 3 pontes), validação de integridade e as 6 perguntas de negócio |
| `job.yaml` | Export do Databricks Workflow (3 tasks, dependências e agendamento) |
| `print_job_sucesso.png` | Print da execução bem-sucedida do Job com o grafo de dependências |

## Orquestração (Databricks Workflow)

`to_Bronze` → `to_Silver` → `to_Gold` (cada task só inicia após a conclusão da anterior).
Agendamento diário às 06:00 (America/Sao_Paulo), compute serverless. Parâmetros por *widgets* com valor padrão, então os notebooks rodam sem intervenção.
Como o modo `append` acrescenta dados a cada execução, a Silver mantém sempre a versão mais recente de cada registro (via `ingestion_datetime`), e o pipeline é seguro para reexecuções.

## Como rodar
1. No Databricks: **Catalog → schema → Create → Volume** (`inputs`) e faça upload dos 5 CSVs da pasta Inputs.
2. Importe os 3 `.ipynb` no Workspace (ajuste o widget `catalogo` se não for `workspace`).
3. Crie o Job (Workflows → Create job) com as 3 tasks e as dependências, ou rode os notebooks na ordem.

## Mock da API do Banco Central
O endpoint PTAX está instável, então o notebook Landing_to_Bronze traz o widget **`usar_mock`** (padrão `true`):
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
