# RAG (Retrieval-Augmented Generation) — Guida completa

## Cos'è il RAG

RAG sta per **Retrieval-Augmented Generation**. È una tecnica che migliora le risposte di un LLM (Large Language Model) fornendogli, al momento della chiamata, solo i **frammenti di documentazione più rilevanti** per la domanda — invece di includere tutto il testo nel prompt.

Il problema che risolve: i documenti tecnici possono essere lunghi centinaia di pagine. Includere tutto nel prompt supera il context window dell'LLM, è costoso in token e introduce rumore. RAG seleziona solo le parti pertinenti.

---

## Come funziona: flusso step-by-step

```
FASE OFFLINE (una volta sola, quando si caricano i documenti)
──────────────────────────────────────────────────────────────
Documenti (.docx / .pdf)
        │
        ▼
  Conversione in testo   ←── transform_docx_to_context() / pdf_to_text()
        │
        ▼
  Testo grezzo concatenato
        │
        ▼
  Chunking (suddivisione in frammenti)   ←── RecursiveCharacterTextSplitter
        │
        ▼
  Embedding di ogni chunk  ←── FastEmbedderWrapper (modello BAAI/bge-small-en)
        │
        ▼
  Vector Store in memoria   ←── InMemoryVectorStore


FASE ONLINE (per ogni cluster / query)
──────────────────────────────────────
  Query (es. sommario analitico del cluster)
        │
        ▼
  Embedding della query
        │
        ▼
  Ricerca per similarità nel vector store  ←── similarity_search(query, k=5)
        │
        ▼
  Top-k chunk più simili
        │
        ▼
  Inserimento nel prompt LLM come contesto
        │
        ▼
  Risposta LLM arricchita con info pertinenti
```

---

## Dipendenze richieste

```toml
# pyproject.toml / requirements
fastembed          # embedding leggero, no GPU, scarica il modello automaticamente
langchain-core     # InMemoryVectorStore, interfaccia Embeddings
langchain-text-splitters   # RecursiveCharacterTextSplitter
python-docx        # lettura file .docx
pymupdf4llm        # conversione PDF → Markdown (include tabelle)
wordninja          # segmentazione parole composte (post-processing PDF)
```

---

## Funzioni del progetto

### `rag_utils.py`

---

#### `FastEmbedderWrapper`

```python
class FastEmbedderWrapper(Embeddings):
    def __init__(self, embedding_model: TextEmbedding): ...
    def embed_documents(self, texts: list[str]) -> list[list[float]]: ...
    def embed_query(self, text: str) -> list[float]: ...
```

**Cosa fa:**  
Adattatore che espone il modello di embedding [FastEmbed](https://github.com/qdrant/fastembed) (BAAI/bge-small-en-v1.5, ~130 MB) attraverso l'interfaccia `Embeddings` di LangChain. Questo permette di usarlo direttamente con `InMemoryVectorStore` o qualsiasi altro vector store LangChain.

- `embed_documents` — converte una lista di testi in una lista di vettori (usato durante l'indicizzazione dei chunk).
- `embed_query` — converte una singola stringa query in un vettore (usato durante la ricerca per similarità).

---

#### `_get_embedding_model()`

```python
def _get_embedding_model() -> FastEmbedderWrapper:
```

**Cosa fa:**  
Gestisce un singleton a livello di modulo: il modello di embedding viene caricato **una volta sola** alla prima chiamata e riutilizzato per tutte le chiamate successive. Evita di ricaricare i pesi su disco ad ogni creazione di vector store (operazione lenta).

---

#### `create_in_memory_rag(raw_text: str) -> InMemoryVectorStore`

```python
def create_in_memory_rag(raw_text: str) -> InMemoryVectorStore:
```

**Cosa fa:**  
Funzione principale per costruire il RAG. Riceve un testo grezzo (già estratto dai documenti), lo suddivide in chunk e lo indicizza in un vector store in memoria.

**Parametri:**
- `raw_text` — stringa con tutto il testo della documentazione (output di `load_llm_context_files`).

**Restituisce:**
- `InMemoryVectorStore` pronto per chiamate `.similarity_search(query, k=N)`.

**Configurazione chunk:**
| Parametro | Valore | Significato |
|---|---|---|
| `chunk_size` | 2000 caratteri | Lunghezza massima di ogni frammento |
| `chunk_overlap` | 400 caratteri | Sovrapposizione tra chunk consecutivi (preserva contesto ai bordi) |

**Esempio d'uso:**
```python
from danieli_anomaly_detection.rag_utils import create_in_memory_rag

vector_db = create_in_memory_rag(raw_text)

# Recupera i 5 chunk più rilevanti
docs = vector_db.similarity_search("anomalia pressione idraulica", k=5)
context = "\n\n".join(doc.page_content for doc in docs)
```

---

### `LLM_utils.py` — Conversione documenti

---

#### `load_llm_context_files(llm_context_dir: str | None) -> str | None`

```python
def load_llm_context_files(llm_context_dir: str | None) -> str | None:
```

**Cosa fa:**  
Funzione di orchestrazione: legge **tutti** i file `.docx` e `.pdf` in una directory e li concatena in un'unica stringa di testo, pronta per essere passata a `create_in_memory_rag`.

**Parametri:**
- `llm_context_dir` — percorso della directory contenente i documenti. Se `None` o inesistente, restituisce `None`.

**Comportamento:**
1. Cerca tutti i `.docx` → chiama `transform_docx_to_context` su ognuno.
2. Cerca tutti i `.pdf` → chiama `pdf_to_text` su ognuno.
3. Ogni documento viene prefissato con `### nome_file` per distinguerli.
4. Restituisce la concatenazione con `\n\n` tra i documenti.

**Esempio d'uso:**
```python
machine_context = load_llm_context_files("config/llm_context/")
vector_db = create_in_memory_rag(machine_context or "")
```

---

#### `transform_docx_to_context(docx_path) -> str`

```python
def transform_docx_to_context(docx_path) -> str:
```

**Cosa fa:**  
Converte un file `.docx` in testo puro adatto al consumo da parte di un LLM. Itera su paragrafi e tabelle in **ordine di documento** (non solo testo, ma anche tabelle embedded).

**Dettagli implementativi:**
- Usa `iter_block_items` per rispettare l'ordine fisico di paragrafi e tabelle nel file Word.
- I paragrafi vuoti vengono saltati.
- Le tabelle vengono convertite in Markdown tramite `table_to_markdown`.

**Dipendenza:** `python-docx`

---

#### `iter_block_items(parent)`

```python
def iter_block_items(parent):
```

**Cosa fa:**  
Generatore che itera su tutti i blocchi figlio (paragrafi e tabelle) di un documento Word, **nel loro ordine fisico** nel file XML. Necessario perché `python-docx` espone `.paragraphs` e `.tables` separatamente, perdendo l'ordine originale.

---

#### `table_to_markdown(table) -> str`

```python
def table_to_markdown(table) -> str:
```

**Cosa fa:**  
Converte una tabella `python-docx` in una stringa Markdown con pipe (`|`). I caratteri `|` nelle celle vengono sostituiti con `/` per non rompere la struttura della tabella.

**Output esempio:**
```markdown
| Componente | Pressione nominale | Soglia allarme |
| --- | --- | --- |
| Pompa A | 250 bar | 220 bar |
```

---

#### `pdf_to_text(pdf_path: Path) -> str`

```python
def pdf_to_text(pdf_path: Path) -> str:
```

**Cosa fa:**  
Converte un file PDF in testo Markdown (incluse le tabelle) usando `pymupdf4llm`. Non usa modelli ML — carica in pochi millisecondi. Le tabelle vengono preservate come pipe-table Markdown.

**Dipendenza:** `pymupdf4llm` (wrapper attorno a PyMuPDF / fitz)

**Post-processing:** chiama `_post_process_pdf_text` per correggere artefatti comuni dell'estrazione PDF.

---

#### `_post_process_pdf_text(text: str) -> str`

```python
def _post_process_pdf_text(text: str) -> str:
```

**Cosa fa:**  
Corregge artefatti tipici dei PDF estratti da `pymupdf4llm`:

| Problema | Fix applicato |
|---|---|
| Righe di tabella corrotte (solo `\|`) | Rimosse con regex |
| Bullet `- ` dentro blocchi Mermaid | Rimossi con regex |
| Mancanza spazi ai confini cifra-lettera | Inseriti con regex `(\d)([a-z])` |
| Parole camelCase unite | Separate con regex `([a-z])([A-Z])` |
| Parole composte tutto-minuscolo | Segmentate con `wordninja` (opzionale) |

---

## Come usare RAG in un nuovo progetto

### Installazione

```bash
pip install fastembed langchain-core langchain-text-splitters python-docx pymupdf4llm
# opzionale per post-processing PDF:
pip install wordninja
```

### Codice minimale completo

```python
from pathlib import Path
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_text_splitters import RecursiveCharacterTextSplitter
from fastembed import TextEmbedding
from langchain_core.embeddings import Embeddings


# 1. Wrapper FastEmbed → LangChain
class FastEmbedderWrapper(Embeddings):
    def __init__(self):
        self.model = TextEmbedding()  # scarica automaticamente BAAI/bge-small-en-v1.5

    def embed_documents(self, texts):
        return [list(e) for e in self.model.embed(texts)]

    def embed_query(self, text):
        return list(next(iter(self.model.embed([text]))))


# 2. Costruisci il vector store da testo grezzo
def create_vector_store(raw_text: str) -> InMemoryVectorStore:
    embedder = FastEmbedderWrapper()
    splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=400)
    chunks = splitter.split_text(raw_text)
    return InMemoryVectorStore.from_texts(chunks, embedder)


# 3. Converti i tuoi documenti in testo
# Per .docx:
from docx import Document

def docx_to_text(path: Path) -> str:
    doc = Document(path)
    return "\n".join(p.text for p in doc.paragraphs if p.text.strip())

# Per .pdf:
import pymupdf4llm

def pdf_to_text(path: Path) -> str:
    result = pymupdf4llm.to_markdown(str(path))
    return result if isinstance(result, str) else "\n\n".join(result)


# ── UTILIZZO ──────────────────────────────────────────────────────────────────

# Carica e concatena i documenti
docs_text = docx_to_text(Path("manuale.docx")) + "\n\n" + pdf_to_text(Path("specs.pdf"))

# Costruisci il RAG
vector_db = create_vector_store(docs_text)

# Interroga: recupera i 5 chunk più rilevanti per la query
query = "Come si diagnostica un'anomalia alla pompa idraulica?"
docs = vector_db.similarity_search(query, k=5)

retrieved_context = "\n\n".join(
    f"=== SEZIONE {i+1} ===\n{doc.page_content}"
    for i, doc in enumerate(docs)
)

# Passa il contesto all'LLM
from openai import AzureOpenAI

client = AzureOpenAI(api_key="...", api_version="2024-06-01", azure_endpoint="...")
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Sei un esperto di manutenzione industriale."},
        {"role": "user", "content": f"Contesto documentazione:\n{retrieved_context}\n\nDomanda: {query}"},
    ],
    temperature=0,
    max_completion_tokens=500,
)
print(response.choices[0].message.content)
```

---

## Note pratiche

- **`InMemoryVectorStore`** non persiste su disco — va ricostruito ad ogni avvio. Per uso in produzione con molti documenti, considera `Chroma` o `FAISS` con persistenza.
- **`chunk_size=2000` + `chunk_overlap=400`** è un buon punto di partenza. Riduci `chunk_size` se i documenti sono molto tecnici e densi (più precisione), aumentalo se sono narrativi (più contesto per chunk).
- **`k=5`** nel `similarity_search` recupera 5 chunk. Aumenta se le risposte dell'LLM sembrano incomplete, riduci se superi il context window.
- Il modello FastEmbed (`BAAI/bge-small-en-v1.5`) viene scaricato automaticamente alla prima esecuzione (~130 MB, in `~/.cache/fastembed`). Non richiede GPU.
- Per documenti in italiano, valuta `BAAI/bge-m3` (multilingua) al posto di `bge-small-en`.
