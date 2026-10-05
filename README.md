# 🧠 DocMind RAG — Energia no Brasil

**FIAP · Ciência da Computação · Turma 1CCPK · Prompt Engineering and Artificial Intelligence · CKP02 (Módulo 2)**
Prof. Jorge Luiz Gomes

## Integrantes

| Nome | RM |
|---|---|
| Daniel Mana | 571632 |
| Davi Queiroz Zuolo | 571669 |
| Gustavo Zagato | 569420 |
| Kayo Henderson | 570706 |

## Sobre o projeto

Assistente de perguntas e respostas sobre documentos **oficiais** do setor de energia brasileiro: regulação da geração
distribuída (ANEEL), matriz energética (EPE), planejamento decenal (MME/EPE) e potencial solar (INPE). O sistema recupera
os trechos mais relevantes e gera respostas fundamentadas, sempre citando documento e página de origem.

**Pipeline:** `load (PyMuPDF) → split (RecursiveCharacterTextSplitter) → embed (nomic-embed-text) → store (ChromaDB) → retrieve (+ filtro de metadados + reranking) → generate (gemma4:cloud)`

| Componente | Escolha |
|---|---|
| Embeddings | `nomic-embed-text` via Ollama Cloud (`OllamaEmbeddings`, `embed_documents` / `embed_query`) |
| LLM | `gemma4:cloud` via Ollama Cloud, `temperature=0`, `OLLAMA_API_KEY` como Secret |
| Vector store | ChromaDB local (`PersistentClient`), distância cosseno, coleções `energia_no_brasil_cs512` e `energia_no_brasil_cs1024` |
| Chunking comparado | `RecursiveCharacterTextSplitter` com separadores `["\n\n", "\n", ". ", " ", ""]`, `chunk_size` 512 e 1024, overlap de 12,5% |
| Avaliação | RAGAS (`faithfulness`, `answer_relevancy`) com 8 perguntas que cobrem os 6 documentos + taxa de recuperação da fonte certa |
| Diferenciais | Metadata filtering (`where=` por `tipo` e `categoria`), reranking com `cross-encoder/ms-marco-MiniLM-L-6-v2`, interface Gradio |

## Base de conhecimento

| Arquivo | Documento | Tipo | Categoria | Fonte |
|---|---|---|---|---|
| `ren_aneel_1059_2023.pdf` | Resolução Normativa ANEEL nº 1.059/2023 (regulamenta a Lei 14.300/2022) | resolução | regulacao | [ANEEL](http://www2.aneel.gov.br/cedoc/ren20231059.pdf) |
| `ben_sintese_2026.pdf` | Balanço Energético Nacional 2026 — Relatório Síntese (ano base 2025) | relatório | matriz_energetica | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/balanco-energetico-nacional-ben) |
| `pde2034_plano_aprovado.pdf` | Plano Decenal de Expansão de Energia 2034 | plano | planejamento | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/plano-decenal-de-expansao-de-energia-2034) |
| `pde2034_caderno_demanda_eletricidade.pdf` | PDE 2034 — Caderno de Demanda de Eletricidade | estudo | demanda | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/plano-decenal-de-expansao-de-energia-2034) |
| `pde2034_caderno_demanda_transportes.pdf` | PDE 2034 — Caderno de Demanda Energética do Setor de Transportes | estudo | demanda | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/plano-decenal-de-expansao-de-energia-2034) |
| `atlas_energia_solar_2017.pdf` | Atlas Brasileiro de Energia Solar — 2ª edição (INPE/LABREN) | atlas | recurso_solar | [INPE](https://www.absolar.org.br/wp-content/uploads/2021/02/Atlas_Brasileiro_Energia_Solar_2a_Edicao.pdf) |

São cerca de 840 páginas com texto extraível. Todos os metadados estão em [`fontes.csv`](fontes.csv).

## Estrutura do repositório

```
├── CKP02_Energia_grupo.ipynb   # notebook completo (exportado já executado)
├── README.md
├── fontes.csv                  # manifesto da base: título, tipo, categoria, data e link de cada documento
└── docs/                       # documentos da base (PDF com texto, TXT ou MD)
```

## Como executar

1. Abra o notebook no Google Colab.
2. Em 🔑 **Secrets**, crie `OLLAMA_API_KEY` com a sua chave do Ollama Cloud e libere o acesso ao notebook
   (os secrets do Colab são por usuário: quem executa precisa cadastrar a própria chave).
3. *Runtime → Run all*. O notebook clona este repositório (`docs/` + `fontes.csv`), indexa, avalia com RAGAS e,
   no fim, abre a interface Gradio com uma URL pública temporária.

Os resultados (tabelas por pergunta, gráfico, configuração vencedora, efeito do reranking e análise de falha) ficam nas
Seções 8 e 9 do notebook executado.

## Como adicionar novos documentos à base

**Opção A — reindexar tudo (recomendada):**
1. Coloque o arquivo em `docs/`. Aceita PDF com texto selecionável, TXT ou MD (PDF escaneado não funciona sem OCR).
2. Acrescente uma linha no `fontes.csv`:
   ```csv
   arquivo,titulo,tipo,categoria,data,url_fonte
   novo_documento.pdf,Título oficial do documento,estudo,demanda,2025-03-10,https://link-oficial
   ```
   `tipo` e `categoria` alimentam o filtro de metadados (`where=`), então reutilize os valores já existentes quando fizer sentido.
3. Faça commit no repositório e rode o notebook do início (*Run all*). A mudança no `fontes.csv` é detectada e as coleções
   são recriadas automaticamente.

**Opção B — incremental, dentro de uma sessão já executada:**
```python
adicionar_documento("/content/novo_documento.pdf", "Título oficial", "estudo",
                    "demanda", "2025-03-10", "https://link-oficial")
```
A função copia o arquivo para `docs/`, registra no `fontes.csv` e indexa apenas os chunks novos em todas as coleções.

## Uso no CKP03

`buscar(consulta, k, filtro, rerank)` foi escrita para ser envolvida com `@tool` no agente do CKP03.
`responder(pergunta)` devolve a resposta e os trechos de origem.

## Decisões técnicas

- **Curadoria:** a versão em inglês da Síntese do BEN (duplicata), o BEN completo (tabelas, redundante com a Síntese) e a
  planilha da matriz ficaram fora da base. A Lei 14.300/2022 é coberta pela REN 1.059/2023, que a regulamenta.
- **Chunk 256 só na estatística:** nesta base ele gera quase o dobro de chunks do 512, com média abaixo de 200 caracteres
  (fragmentos de tabela sem contexto). Indexá-lo dobraria as chamadas de embedding sem chance real de vencer.
- **Versões fixadas:** `ragas==0.4.3` e `langchain-community==0.4.1`. A versão 0.4.2 do `langchain-community` removeu um
  módulo que o `ragas` importa e quebra a avaliação.
- **Reserva do RAGAS:** se o juiz não devolver um formato válido em alguma pergunta, aquele score é calculado com uma
  implementação equivalente (mesmo algoritmo do paper). A coluna `metodo` indica a origem de cada número.
