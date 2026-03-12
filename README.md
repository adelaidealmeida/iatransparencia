# HyPA-RAG — Transparência Inteligente

Sistema de Recuperação Aumentada por Geração (RAG) híbrido para consulta inteligente de atos normativos do Poder Executivo do Estado do Pará. Desenvolvido como parte da pesquisa submetida ao **CSBC 2026**.

---

## Visão Geral

O HyPA-RAG combina busca densa (vetorial) e esparsa (BM25) com reranking neural para responder perguntas em linguagem natural sobre documentos jurídicos e normativos, com rastreabilidade de fonte por chunk.

```
Pergunta → Classificação de complexidade → Retrieval híbrido (KNN + BM25)
         → Fusão RRF → Reranking (cross-encoder) → Geração (LLaMA 3.1)
         → Resposta com citações [Fonte: ... | Chunk: ...]
```

---

## Pré-requisitos

| Componente | Versão testada | Observação |
|---|---|---|
| Python | 3.11+ | |
| Redis Stack | 7.2+ | Requer módulo RediSearch |
| LM Studio | 0.3+ | Serve os modelos localmente via API OpenAI-compatible |
| CUDA (opcional) | 11.8+ | CPU funciona, mas é mais lento para embeddings e reranker |

### Modelos necessários no LM Studio

Baixe e carregue os seguintes modelos no LM Studio antes de rodar:

| Papel | Modelo |
|---|---|
| Gerador de respostas | `meta-llama-3.1-8b-instruct` |
| Avaliador (juiz LLM) | `qwen2.5-7b-instruct` |

Os modelos de embedding e reranker são baixados automaticamente pelo Hugging Face na primeira execução:

| Papel | Modelo |
|---|---|
| Embeddings | `intfloat/e5-base-v2` (768 dims) |
| Reranker | `cross-encoder/ms-marco-MiniLM-L-6-v2` |

---

## Instalação

### 1. Clonar o repositório

```bash
git clone https://github.com/seu-usuario/hypa-rag.git
cd hypa-rag
```

### 2. Criar ambiente virtual e instalar dependências

```bash
python -m venv .venv
source .venv/bin/activate        # Linux/macOS
# .venv\Scripts\activate         # Windows

pip install -r requirements.txt
```

### 3. Instalar e iniciar o Redis Stack

**Via Docker (recomendado):**
```bash
docker run -d --name redis-stack \
  -p 6379:6379 \
  redis/redis-stack-server:latest
```

**Via apt (Ubuntu):**
```bash
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt update && sudo apt install redis-stack-server
sudo systemctl start redis-stack-server
```

### 4. Iniciar o LM Studio

Abra o LM Studio, carregue os dois modelos listados acima e certifique-se de que o servidor local está ativo em `http://localhost:1234`.

---

## Dependências Python

Crie um arquivo `requirements.txt` com:

```
numpy
redis[hiredis]
requests
pypdf
sentence-transformers
tqdm
flask
```

---

## Estrutura do Repositório

```
hypa-rag/
├── hyparag_unificado_28_limpo.ipynb   # Notebook principal
├── eval_set_auto.jsonl                # Dataset de avaliação (10 perguntas)
├── eval_set_transparencia.jsonl       # Dataset de candidatos
├── documentos/                        # PDFs a indexar (ver nota abaixo)
│   └── .gitkeep
├── .gitignore
└── README.md
```

> **Nota sobre os PDFs:** Os documentos normativos utilizados nos experimentos são públicos e estão disponíveis no Portal da Transparência do Estado do Pará. A pasta `documentos/` deve ser populada manualmente antes da indexação.

---

## Como Usar

### 1. Adicionar os PDFs

Coloque os arquivos `.pdf` na pasta `documentos/`:

```bash
mkdir -p documentos
cp /caminho/dos/seus/pdfs/*.pdf documentos/
```

### 2. Abrir o notebook

```bash
jupyter notebook hyparag_unificado_28_limpo.ipynb
```

### 3. Executar as células em ordem

As células estão organizadas em dois pipelines:

**Pipeline A — Servidor Flask (células 1–9)**
Configura o ambiente, carrega modelos, define o pipeline de retrieval, métricas e sobe o servidor Flask em `http://localhost:5000`.

**Pipeline B — Reindexação (células 10–15)**
Processa os PDFs da pasta `documentos/`, indexa os chunks no Redis e inicia o Flask. É o pipeline canônico de indexação — use este para reindexar após adicionar novos documentos.

A célula 15 (`main()`) executa automaticamente a reindexação incremental e sobe o Flask.

### 4. Acessar a interface

Após executar a célula 15, o browser abrirá automaticamente em `http://localhost:5000`. Caso não abra, acesse manualmente.

---

## Variáveis de Ambiente

Todas as configurações têm valores padrão e funcionam sem `.env`. Para customizar, exporte as variáveis antes de iniciar o Jupyter:

| Variável | Padrão | Descrição |
|---|---|---|
| `REDIS_HOST` | `localhost` | Host do Redis |
| `REDIS_PORT` | `6379` | Porta do Redis |
| `REDIS_PASSWORD` | _(vazio)_ | Senha do Redis (se houver) |
| `REDIS_INDEX` | `idx:hyparag_v13` | Nome do índice RediSearch |
| `REDIS_PREFIX` | `doc:v13:` | Prefixo das chaves no Redis |
| `LM_API_URL` | `http://localhost:1234/v1/chat/completions` | Endpoint do LM Studio |
| `MODEL_NAME` | `meta-llama-3.1-8b-instruct` | Modelo gerador |
| `EMBEDDING_MODEL_NAME` | `intfloat/e5-base-v2` | Modelo de embeddings |
| `PASTA_PDFS` | `./documentos` | Pasta com os PDFs |
| `MIN_CHARS_CHUNK` | `250` | Tamanho mínimo de chunk (chars) |
| `MAX_CHARS_CHUNK` | `3500` | Tamanho máximo de chunk (chars) |
| `RAG_EVAL_ENABLE` | `1` | Habilita log de métricas (`0` desabilita) |
| `RAG_EVAL_LOG` | `rag_metrics_log.jsonl` | Arquivo de log de métricas |
| `OUT_DIR` | `./reindex_v13_out` | Pasta de auditoria da reindexação |

---

## Pipeline de Indexação

Os PDFs são processados pela seguinte cadeia:

1. **Extração** — PyPDF extrai texto página a página; PDFs com menos de 300 caracteres são descartados (OCR/escaneados)
2. **Normalização** — desfaz hifenização, normaliza quebras de linha e espaços
3. **Chunking por artigo jurídico** — divide pelo padrão `Art. Nº`; fallback por `TÍTULO/CAPÍTULO/SEÇÃO`; último recurso: texto integral
4. **Pós-processamento** — chunks menores que 250 chars são descartados; chunks maiores que 3.500 chars são fatiados por parágrafo
5. **Filtro de qualidade** — chunks com menos de 55% de letras (tabelas/OCR ruidoso) são removidos
6. **Metadados** — extrai tipo (lei, decreto, instrução normativa), número, ano e área temática por regras léxicas
7. **Indexação incremental** — manifesto JSON controla quais PDFs já foram indexados (por tamanho e data de modificação)
8. **Redis HNSW** — chunks gravados como hashes com embedding FLOAT32 (768 dims), índice HNSW com M=16, EF_CONSTRUCTION=200

---

## Pipeline de Retrieval

Para cada pergunta:

1. **Classificação de complexidade** — `simples`, `media` ou `complexa` (ajusta k_dense, k_sparse, k_final)
2. **Detecção de filtros** — identifica área temática, tipo normativo, ano e documento específico na pergunta
3. **Busca densa** — KNN vetorial com filtros TAG no RediSearch
4. **Busca esparsa** — BM25 full-text com os mesmos filtros
5. **Fusão RRF** — Reciprocal Rank Fusion com pesos [2.0 dense, 1.0 sparse]
6. **Reranking** — cross-encoder `ms-marco-MiniLM-L-6-v2` reordena os candidatos
7. **Geração** — LLaMA 3.1 8B gera resposta em pt-BR com citações por chunk

### Áreas temáticas suportadas

`organizacao_administrativa` · `processo_administrativo` · `servidor_publico` · `educacao` · `licitacoes_contratos` · `financas_publicas` · `transparencia_controle` · `instrumentos_planejamento` · `norma_geral` · `outros`

---

## Avaliação

O notebook inclui um pipeline completo de avaliação com métricas RAGAS:

| Métrica | Resultado (n=10) |
|---|---|
| Hit Rate | 1.0000 |
| MRR | 1.0000 |
| nDCG | 1.0000 |
| Context Recall | 1.0000 |
| Faithfulness | 0.8100 |
| Answer Relevance | 0.8608 |
| Answer Correctness | 0.7700 |

Para rodar a avaliação:

```python
# Célula 17 do notebook
avaliar_dataset(
    eval_path="eval_set_auto.jsonl",
    out_path="eval_results.jsonl"
)
```

Os resultados são gravados em `eval_results.jsonl`. A célula 18 calcula e exibe as médias por métrica.

---

## Arquivos Gerados (não versionados)

| Arquivo/Pasta | Descrição |
|---|---|
| `reindex_v13_out/` | Manifesto, auditoria e sumário da reindexação |
| `rag_metrics_log.jsonl` | Log de métricas por consulta em produção |
| `eval_results.jsonl` | Resultados da avaliação do dataset |
| `eval_candidatos.json` | Candidatos gerados para análise do eval_set |

---

## Citação

Se você usar este sistema em sua pesquisa, por favor cite:

```
@inproceedings{hyparag2026,
  title     = {Transparência Inteligente: Um Sistema de Recuperação Aumentada
               por Geração para Transparência Pública de Atos Normativos},
  booktitle = {Anais do CSBC 2026},
  year      = {2026}
}
```

---

## Licença

Este projeto é disponibilizado para fins acadêmicos. Consulte o arquivo `LICENSE` para detalhes.
