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

## Entrega

| Item | Onde |
|---|---|
| Notebook executado (`.ipynb`) | `CKP02_DocMindRAG_Energy.ipynb` |
| **Link do Colab** (permissão de leitura) | ⚠️ **PREENCHER ANTES DE ENVIAR** |
| Documentos da base | `docs/` — 6 PDFs oficiais |
| Manifesto da base | `fontes.csv` |
| Repositório público usado pelo notebook | https://github.com/DaviZuolo07/DocMind_RAG |

## Sobre o projeto

Assistente de perguntas e respostas sobre documentos **oficiais** do setor de energia brasileiro: regulação da geração
distribuída (ANEEL), matriz energética (EPE), planejamento decenal (MME/EPE) e potencial solar (INPE). O sistema recupera
os trechos mais relevantes e gera respostas fundamentadas, sempre citando documento e página de origem.

**Pipeline:** `load (PyMuPDF) → split (RecursiveCharacterTextSplitter) → embed (nomic-embed-text) → store (ChromaDB) → retrieve (+ filtro de metadados + reranking) → generate (gemma4:cloud)`

| Componente | Escolha |
|---|---|
| Embeddings | `nomic-embed-text` via `OllamaEmbeddings` (`embed_documents` / `embed_query`), 768 dimensões — ver nota sobre o host |
| LLM (generate e juiz do RAGAS) | `gemma4:cloud` via **Ollama Cloud**, `temperature=0`, `OLLAMA_API_KEY` como Secret do Colab |
| Vector store | ChromaDB local (`PersistentClient`), distância cosseno, coleções `energia_no_brasil_cs512` e `energia_no_brasil_cs1024` |
| Chunking comparado | `RecursiveCharacterTextSplitter`, separadores `["\n\n", "\n", ". ", " ", ""]`, `chunk_size` 512 e 1024, `chunk_overlap` de 12,5% (64 e 128) |
| Recuperação | `K_FINAL = 6` trechos finais, 12 candidatos antes do reranking |
| Avaliação | RAGAS (`faithfulness`, `answer_relevancy`) com 8 perguntas que cobrem os 6 documentos + taxa de recuperação da fonte certa |
| Diferenciais | Metadata filtering (`where=` por `tipo` e `categoria`), reranking com `cross-encoder/ms-marco-MiniLM-L-6-v2`, interface Gradio |

> ⚙️ **Nota sobre o host dos embeddings.** O endpoint `/api/embed` do Ollama Cloud devolveu `401` mesmo com a chave
> válida. Para não trocar de modelo, o notebook sobe um servidor **Ollama local dentro do próprio Colab** e roda ali o
> **mesmo `nomic-embed-text`** exigido pelo CKP02 (função `iniciar_ollama_local()`, Seção 2.2). O modelo e a API
> (`OllamaEmbeddings`, `embed_query` / `embed_documents`) são exatamente os pedidos; só o host muda. O `gemma4:cloud`
> continua no Ollama Cloud com a chave como Secret.

## Base de conhecimento

6 documentos oficiais, **839 páginas com texto extraível**, ~1,83 milhão de caracteres.

| Arquivo | Documento | `tipo` | `categoria` | Páginas | Fonte |
|---|---|---|---|---|---|
| `ren_aneel_1059_2023.pdf` | Resolução Normativa ANEEL nº 1.059/2023 (regulamenta a Lei 14.300/2022) | `resolucao` | `regulacao` | 45 | [ANEEL](http://www2.aneel.gov.br/cedoc/ren20231059.pdf) |
| `ben_sintese_2026.pdf` | Balanço Energético Nacional 2026 — Relatório Síntese (ano base 2025) | `relatorio` | `matriz_energetica` | 73 | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/balanco-energetico-nacional-ben) |
| `pde2034_plano_aprovado.pdf` | Plano Decenal de Expansão de Energia 2034 (MME/EPE) | `plano` | `planejamento` | 576 | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/plano-decenal-de-expansao-de-energia-2034) |
| `pde2034_caderno_demanda_eletricidade.pdf` | PDE 2034 — Caderno de Demanda de Eletricidade | `estudo` | `demanda` | 18 | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/plano-decenal-de-expansao-de-energia-2034) |
| `pde2034_caderno_demanda_transportes.pdf` | PDE 2034 — Caderno de Demanda Energética do Setor de Transportes | `estudo` | `demanda` | 48 | [EPE](https://www.epe.gov.br/pt/publicacoes-dados-abertos/publicacoes/plano-decenal-de-expansao-de-energia-2034) |
| `atlas_energia_solar_2017.pdf` | Atlas Brasileiro de Energia Solar — 2ª edição (INPE/LABREN) | `atlas` | `recurso_solar` | 79 | [INPE/LABREN](https://www.absolar.org.br/wp-content/uploads/2021/02/Atlas_Brasileiro_Energia_Solar_2a_Edicao.pdf) |

Os valores de `tipo` e `categoria` acima são **exatamente** os usados nos filtros `where=` (sem acento, minúsculos).
Título, data e link completos de cada documento estão em [`fontes.csv`](fontes.csv).

## Resultados

<!-- RESULTADOS:INICIO -->

**Chunking:**

| Config | `chunk_size` | `chunk_overlap` | Nº de chunks | Tamanho médio | Indexada? |
|---|---|---|---|---|---|
| Recursive-256 | 256 | 32 | 9 583 | 191 | não (ver nota) |
| Recursive-512 | 512 | 64 | 4 836 | 384 | sim |
| Recursive-1024 | 1024 | 128 | 2 792 | 690 | sim |

**RAGAS — 8 perguntas, sem reranking (só o `chunk_size` muda):**

| Config | faithfulness médio | answer_relevancy médio | Fonte certa recuperada |
|---|---|---|---|
| Recursive-512 | 0.698 | 0.541 | 100% |
| Recursive-1024 🏆 | 0.771 | 0.637 | 100% |

**Efeito do reranking (diferencial), sobre a configuração vencedora:**

| Versão | faithfulness | answer_relevancy | Fonte certa recuperada |
|---|---|---|---|
| Recursive-1024 sem reranking | 0.771 | 0.637 | 100.0% |
| Recursive-1024 + cross-encoder | 0.792 | 0.784 | 87.5% |

**Vencedora: Recursive-1024** · faithfulness médio 0.771 (acima da meta de 0,7) · base de 839 páginas e 1 825 861 caracteres

<!-- RESULTADOS:FIM -->

A última célula da Seção 8 do notebook imprime esse bloco já preenchido com os números da execução. Depois de cada
*Run all*, copie a saída e cole aqui entre as marcas `RESULTADOS:INICIO` / `RESULTADOS:FIM` — assim o README nunca
diverge do notebook.

### Leitura dos números

**Recursive-1024 venceu** pelo maior faithfulness médio, critério prioritário porque mede se a resposta está ancorada
nos documentos. Nesta base a informação costuma vir com condições e exceções no mesmo parágrafo (limites de
micro/minigeração por fonte, prazos de crédito); chunks de 1024 caracteres mantêm a regra inteira num só trecho,
enquanto o 512 corta a regra ao meio e o modelo tende a completar o resto. A pergunta mais sensível ao tamanho foi a do
PERS (0,50 no 512 → 1,00 no 1024).

**O reranking fica ligado** no pipeline final: o cross-encoder lê pergunta e trecho juntos e subiu as duas métricas, ao
custo de uma pergunta em que o documento esperado saiu do top-k.

**Limite reconhecido:** o faithfulness médio passa a meta de 0,7, mas fica abaixo da zona ideal de 0,9. A análise da
pior pergunta (Seção 9) mostra que o trecho certo *é* recuperado e a resposta final está correta — o juiz é que não
casa todas as afirmações com o contexto. Daí as duas mudanças já incorporadas ao pipeline: `K_FINAL` de 4 para 6 e um
prompt que exige a resposta direta na primeira frase, com números copiados literalmente do trecho.

Tabelas por pergunta, gráfico comparativo, conclusão gerada a partir dos dados e análise de falha estão nas **Seções 8
e 9** do notebook. Os números brutos são salvos em `resultados_ragas.csv` durante a execução.

## Estrutura do repositório

```
DocMind_RAG_1CCPK/
├── CKP02_DocMindRAG_Energy.ipynb   # notebook completo, exportado já executado
├── README.md                       # este arquivo
├── fontes.csv                      # manifesto: arquivo, título, tipo, categoria, data e link de cada documento
└── docs/                           # base de conhecimento (PDF com texto, TXT ou MD)
    ├── ren_aneel_1059_2023.pdf
    ├── ben_sintese_2026.pdf
    ├── pde2034_plano_aprovado.pdf
    ├── pde2034_caderno_demanda_eletricidade.pdf
    ├── pde2034_caderno_demanda_transportes.pdf
    └── atlas_energia_solar_2017.pdf
```

`resultados_ragas.csv` é gerado na raiz durante a execução (Seção 8).

## Como executar

Este notebook usa o **Ollama Cloud** para o `gemma4:cloud`. Os *Secrets* do Colab são por usuário, então quem executar
precisa de uma chave própria — ela não viaja com o notebook compartilhado.

1. Crie uma chave gratuita em <https://ollama.com> → *Settings* → *API keys*.
2. No Colab, ícone 🔑 **Secrets** → *Add new secret* → nome `OLLAMA_API_KEY`, cole a chave e **ative o acesso a este
   notebook** no botão ao lado.
3. *Runtime → Run all*.

O notebook faz o resto sozinho: clona o repositório (`docs/` + `fontes.csv`), sobe o Ollama local para os embeddings,
indexa as duas coleções, avalia com RAGAS e, na Seção 11, publica a interface Gradio com URL pública temporária.

Tempo aproximado num Colab CPU: ~2 min de instalação + ~2 min de indexação (4 836 + 2 792 chunks) + ~3 min de RAGAS.

> Sem a chave, a execução para na Seção 2.2 com uma mensagem dizendo exatamente o que falta. Nenhuma outra célula
> depende de configuração manual.

## Como adicionar novos documentos à base

**Opção A — reindexar tudo (recomendada):**

1. Coloque o arquivo em `docs/`. Aceita **PDF com texto selecionável, TXT ou MD** (PDF escaneado não funciona sem OCR —
   o notebook avisa quando um documento não gera texto).
2. Acrescente uma linha ao `fontes.csv`, seguindo exatamente o cabeçalho:
   ```csv
   arquivo,titulo,tipo,categoria,data,url_fonte
   novo_documento.pdf,Título oficial do documento,estudo,demanda,2025-03-10,https://link-oficial
   ```
   `tipo` e `categoria` alimentam o filtro `where=` — reutilize os valores existentes (`resolucao`, `relatorio`,
   `plano`, `estudo`, `atlas` / `regulacao`, `matriz_energetica`, `planejamento`, `demanda`, `recurso_solar`) sempre que
   couber. `url_fonte` é **obrigatório**: o CKP02 exige fonte citada para cada documento.
3. Faça commit no repositório e rode o notebook do início (*Run all*). O notebook calcula uma assinatura da base; como o
   `fontes.csv` mudou, as duas coleções são recriadas e o RAGAS é recalculado automaticamente.

**Opção B — incremental, dentro de uma sessão já executada:**

```python
adicionar_documento(
    caminho="/content/novo_documento.pdf",
    titulo="Título oficial do documento",
    tipo="estudo",
    categoria="demanda",
    data="2025-03-10",
    url_fonte="https://link-oficial",
)
```

A função copia o arquivo para `docs/`, registra a linha no `fontes.csv` e indexa **apenas os chunks novos** em todas as
coleções (512 e 1024), sem reprocessar o resto da base. Ela recusa formatos não suportados, arquivos já presentes e
chamadas sem `url_fonte`. Reexecute as Seções 8–9 se quiser as métricas atualizadas.

## Uso no CKP03

`buscar(consulta, k=6, filtro=None, rerank=True)` devolve os trechos com texto, metadados e scores — é a função pensada
para ser envolvida com `@tool` no agente LangGraph do CKP03. `responder(pergunta)` roda o pipeline completo
(retrieve → generate) e devolve `{"pergunta", "resposta", "trechos"}`, com a citação `(Fonte: <título>, p. <página>)`
já embutida na resposta. Há também `chain_rag`, versão `Runnable` (LCEL) do pipeline inteiro.

## Decisões técnicas

- **Curadoria da base:** ficaram fora a versão em inglês da Síntese do BEN (duplicata, misturaria idiomas na busca), o
  BEN completo (273 páginas quase só de tabelas, redundante com a Síntese) e a planilha da matriz (formato tabular).
  A Lei 14.300/2022 é coberta pela REN ANEEL nº 1.059/2023, que a regulamenta e traz as mesmas definições.
- **Chunk 256 só na estatística:** gera quase o dobro de chunks do 512, com média de 191 caracteres — fragmentos de
  tabela e legenda sem contexto. Indexá-lo dobraria as chamadas de embedding sem chance real de vencer.
- **Chunks < 60 caracteres descartados:** números de página e rótulos soltos, que só poluem a busca.
- **`K_FINAL = 6`:** subimos de 4 para 6 após a análise da Seção 9. Com k=4 e chunks de 512, a definição de microgeração
  e minigeração — que ocupa dois incisos seguidos da REN 1.059 — às vezes não entrava inteira no contexto.
- **Prompt com resposta direta primeiro:** a instrução de abrir com a resposta em uma frase, copiando números
  literalmente, ataca o `answer_relevancy` baixo medido na primeira rodada, em que as respostas traziam contexto além do
  perguntado.
- **Versões fixadas:** `ragas==0.4.3` e `langchain-community==0.4.1`. A 0.4.2 do `langchain-community` removeu um módulo
  que o `ragas` importa e quebra a avaliação.
- **Reserva do RAGAS:** se o juiz não devolver um formato válido em alguma pergunta, aquele score é recalculado com uma
  implementação equivalente (mesmo algoritmo de Es et al., 2024). A coluna `metodo` da tabela indica a origem de cada
  número.
- **Juiz = modelo gerador:** o RAGAS usa o mesmo `gemma4:cloud` que gera as respostas. É a configuração da Aula 07, mas
  vale registrar o viés: um juiz independente tenderia a ser mais severo.
