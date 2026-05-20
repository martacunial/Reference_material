# RAG (Retrieval-Augmented Generation) — Guida completa

## Cos'è il RAG

RAG sta per **Retrieval-Augmented Generation**. È una tecnica che migliora le risposte di un LLM (Large Language Model) fornendogli, al momento della chiamata, solo i **frammenti di documentazione più rilevanti** per la domanda — invece di includere tutto il testo nel prompt.

**Il problema che risolve:** i documenti tecnici possono essere lunghi centinaia di pagine. Includere tutto nel prompt supera il context window dell'LLM, è costoso in token e introduce rumore. RAG seleziona solo le parti pertinenti.

---

## Come funziona: flusso step-by-step

```
FASE OFFLINE (una volta sola, quando si caricano i documenti)
──────────────────────────────────────────────────────────────
Documenti (.docx / .pdf)
        │
        ▼
  Conversione in testo   ←── transform_docx_to_text() / pdf_to_text()
        │
        ▼
  Testo grezzo concatenato
        │
        ▼
  Chunking (suddivisione in frammenti)   ←── RecursiveCharacterTextSplitter
        │
        ▼
  Embedding di ogni chunk  ←── FastEmbedderWrapper (BAAI/bge-small-en-v1.5)
        │
        ▼
  Vector Store in memoria   ←── InMemoryVectorStore


FASE ONLINE (per ogni query)
──────────────────────────────────────
  Query (testo libero)
        │
        ▼
  Embedding della query
        │
        ▼
  Ricerca per similarità coseno nel vector store  ←── similarity_search(query, k=5)
        │
        ▼
  Top-k chunk più simili
        │
        ▼
  Inserimento nel prompt LLM come contesto
        │
        ▼
  Risposta LLM arricchita con le informazioni pertinenti
```

---

## Dipendenze richieste

```bash
pip install fastembed langchain-core langchain-text-splitters python-docx pymupdf4llm
# opzionale per post-processing PDF (segmentazione parole composte):
pip install wordninja
```

```toml
# oppure in pyproject.toml
dependencies = [
    "fastembed",
    "langchain-core",
    "langchain-text-splitters",
    "python-docx",
    "pymupdf4llm",
    "wordninja",   # opzionale
]
```

---

## File 1 — `rag_utils.py`

Incolla questo file nel tuo progetto. Contiene tutto il necessario per creare il vector store.

```python
"""
RAG utilities.

Provides the embedding model wrapper, module-level singleton management, and
in-memory vector store creation used to retrieve relevant documentation chunks.
"""

import logging
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_text_splitters import RecursiveCharacterTextSplitter
from fastembed import TextEmbedding
from langchain_core.embeddings import Embeddings

logger = logging.getLogger(__name__)


class FastEmbedderWrapper(Embeddings):
    """Adapter that exposes FastEmbed's TextEmbedding through LangChain's
    Embeddings interface, enabling seamless use with LangChain vector stores.

    The underlying model (BAAI/bge-small-en-v1.5, ~130 MB) is downloaded
    automatically on first use to ~/.cache/fastembed. No GPU required.
    """

    def __init__(self, embedding_model: TextEmbedding):
        self.embedding_model = embedding_model

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        """Embed a list of documents. Called during vector store indexing."""
        return [list(e) for e in self.embedding_model.embed(texts)]

    def embed_query(self, text: str) -> list[float]:
        """Embed a single query string. Called during similarity search."""
        return list(next(iter(self.embedding_model.embed([text]))))


# Module-level singleton: the model is loaded once per process and reused across
# all calls to avoid reloading weights (~130 MB) on every vector store creation.
_embedding_model: FastEmbedderWrapper | None = None


def _get_embedding_model() -> FastEmbedderWrapper:
    """Return the shared embedding model, initialising it on first use.

    Uses a module-level singleton so the model weights are loaded only once
    per Python process, regardless of how many times create_in_memory_rag()
    is called.
    """
    global _embedding_model
    if _embedding_model is None:
        _embedding_model = FastEmbedderWrapper(TextEmbedding())
    return _embedding_model


def create_in_memory_rag(raw_text: str) -> InMemoryVectorStore:
    """Create an in-memory RAG vector store from a plain-text string.

    The text is split into overlapping chunks with RecursiveCharacterTextSplitter
    and each chunk is embedded with the shared FastEmbed model (BAAI/bge-small-en-v1.5).
    The resulting vector store supports cosine-similarity search via .similarity_search().

    Parameters
    ----------
    raw_text : str
        Full text to be chunked, embedded, and stored. Typically the concatenated
        output of load_context_files() or any other text extraction pipeline.

    Returns
    -------
    InMemoryVectorStore
        Vector store ready for ``similarity_search(query, k=N)``.
        Returns an empty store (no documents) if raw_text is blank.

    Notes
    -----
    chunk_size=2000, chunk_overlap=400 is a good default for technical documents.
    Reduce chunk_size for very dense/tabular content, increase for narrative text.
    The store lives in RAM and is not persisted to disk. Rebuild on each startup,
    or swap InMemoryVectorStore for Chroma/FAISS if persistence is needed.

    Example
    -------
    >>> vector_db = create_in_memory_rag(raw_text)
    >>> docs = vector_db.similarity_search("hydraulic pump anomaly", k=5)
    >>> context = "\\n\\n".join(doc.page_content for doc in docs)
    """
    embedding_model = _get_embedding_model()

    if not raw_text.strip():
        logger.warning(
            "create_in_memory_rag: input is empty — returning empty vector store."
        )
        return InMemoryVectorStore(embedding=embedding_model)

    text_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=400)
    chunks = text_splitter.split_text(raw_text)

    logger.debug(
        "create_in_memory_rag: indexed %d chunks from %d characters of text.",
        len(chunks),
        len(raw_text),
    )

    return InMemoryVectorStore.from_texts(chunks, embedding_model)
```

---

## File 2 — `doc_utils.py`

Incolla questo file nel tuo progetto. Contiene le funzioni per convertire `.docx` e `.pdf` in testo.

```python
"""
Document-to-text utilities for RAG pipelines.

Converts .docx and .pdf files into plain text / Markdown strings suitable
for chunking and embedding.

Dependencies
------------
python-docx   : pip install python-docx
pymupdf4llm   : pip install pymupdf4llm
wordninja     : pip install wordninja  (optional, improves PDF post-processing)
"""

import re
import logging
from pathlib import Path

from docx import Document
from docx.table import Table
from docx.text.paragraph import Paragraph

logger = logging.getLogger(__name__)


# ── Directory loader ─────────────────────────────────────────────────────────


def load_context_files(context_dir: str | Path | None) -> str | None:
    """Read all .docx and .pdf files in a directory and concatenate their text.

    Iterates the directory, converts each file to plain text, and joins all
    results into a single string separated by double newlines. Each document
    is prefixed with a ``### filename`` header so chunks stay attributable
    after splitting.

    Parameters
    ----------
    context_dir : str | Path | None
        Path to a directory containing .docx and/or .pdf files.
        Returns None if the argument is None, the path does not exist,
        or no supported files are found.

    Returns
    -------
    str | None
        Concatenated text of all documents, or None on failure / empty dir.

    Example
    -------
    >>> text = load_context_files("docs/")
    >>> vector_db = create_in_memory_rag(text or "")
    """
    if not context_dir:
        return None
    context_path = Path(context_dir)
    if not context_path.is_dir():
        return None

    docx_files = sorted(context_path.glob("*.docx"))
    pdf_files = sorted(context_path.glob("*.pdf"))
    print(f"Found {len(docx_files)} .docx files and {len(pdf_files)} .pdf files.")

    if not docx_files and not pdf_files:
        return None

    parts = []
    for f in docx_files:
        try:
            text = transform_docx_to_text(f)
            if text.strip():
                parts.append(f"### {f.stem}\n{text}")
        except Exception as e:
            logger.warning(f"Failed to process {f}: {e}")

    for f in pdf_files:
        try:
            text = pdf_to_text(f)
            if text.strip():
                parts.append(f"### [PDF] {f.stem}\n{text}")
        except Exception as e:
            logger.warning(f"Failed to process PDF {f}: {e}")

    return "\n\n".join(parts) if parts else None


# ── .docx conversion ─────────────────────────────────────────────────────────


def iter_block_items(parent):
    """Yield each paragraph and table child of *parent* in document order.

    python-docx exposes .paragraphs and .tables as separate lists, losing the
    original interleaved order. This generator walks the underlying XML directly
    so tables and paragraphs are yielded in the exact order they appear in the
    document.

    Supports both Document objects and _Cell objects (for nested tables).

    Parameters
    ----------
    parent : Document | _Cell
        A python-docx Document or table cell to iterate over.

    Yields
    ------
    Paragraph | Table
        Each block child in document order.

    Example
    -------
    >>> doc = Document("manual.docx")
    >>> for block in iter_block_items(doc):
    ...     if isinstance(block, Paragraph):
    ...         print(block.text)
    """
    from docx.document import Document as _Document

    if isinstance(parent, _Document):
        parent_elm = parent.element.body
    else:
        parent_elm = parent._element

    for child in parent_elm.iterchildren():
        if child.tag.endswith("}p"):
            yield Paragraph(child, parent)
        elif child.tag.endswith("}tbl"):
            yield Table(child, parent)


def table_to_markdown(table) -> str:
    """Convert a python-docx Table to a GitHub-flavoured Markdown table string.

    Each row becomes a pipe-delimited Markdown row. A separator row is inserted
    after the first row (treated as header). Empty cells are replaced with a
    single space to avoid malformed Markdown. Pipe characters inside cell text
    are replaced with ``/`` to prevent column misalignment.

    Parameters
    ----------
    table : docx.table.Table
        A python-docx Table object, typically obtained from iter_block_items().

    Returns
    -------
    str
        Markdown table string, or an empty string if the table has no rows.

    Example
    -------
    Input table (Word):
        Component | Nominal Pressure | Alarm Threshold
        Pump A    | 250 bar          | 220 bar

    Output:
        | Component | Nominal Pressure | Alarm Threshold |
        | --- | --- | --- |
        | Pump A | 250 bar | 220 bar |
    """
    rows = []
    for row in table.rows:
        cells = []
        for cell in row.cells:
            text = cell.text.strip().replace("\n", " ").replace("|", "/")
            cells.append(text if text else " ")
        formatted_row = "| " + " | ".join(cells) + " |"
        rows.append(formatted_row)

    if not rows:
        return ""

    separator = "| " + " | ".join(["---"] * len(table.rows[0].cells)) + " |"
    rows.insert(1, separator)

    return "\n" + "\n".join(rows) + "\n"


def transform_docx_to_text(docx_path: str | Path) -> str:
    """Convert a .docx file to a plain-text string suitable for LLM consumption.

    Iterates all blocks (paragraphs and tables) in document order using
    iter_block_items(). Paragraphs are reproduced verbatim; tables are converted
    to Markdown pipe-tables via table_to_markdown(). Empty paragraphs are skipped.

    Parameters
    ----------
    docx_path : str | Path
        Path to the .docx file.

    Returns
    -------
    str
        All textual content joined by newlines, ready to be passed to
        create_in_memory_rag() or used directly as LLM prompt context.

    Example
    -------
    >>> text = transform_docx_to_text("technical_manual.docx")
    >>> print(text[:500])
    """
    doc = Document(docx_path)
    full_text = []
    for block in iter_block_items(doc):
        if isinstance(block, Paragraph):
            if block.text.strip():
                full_text.append(block.text)
        elif isinstance(block, Table):
            full_text.append(table_to_markdown(block))
    return "\n".join(full_text)


# ── .pdf conversion ──────────────────────────────────────────────────────────


def pdf_to_text(pdf_path: str | Path) -> str:
    """Convert a PDF to Markdown text (including tables) using pymupdf4llm.

    Uses PyMuPDF under the hood — no ML models required, converts in milliseconds.
    Tables are preserved as Markdown pipe-tables. The output is post-processed
    by _post_process_pdf_text() to fix common extraction artefacts.

    Parameters
    ----------
    pdf_path : str | Path
        Path to the PDF file.

    Returns
    -------
    str
        Markdown-formatted text. Returns an empty string if conversion fails.

    Notes
    -----
    Requires: pip install pymupdf4llm

    Example
    -------
    >>> text = pdf_to_text("datasheet.pdf")
    >>> print(text[:500])
    """
    try:
        import pymupdf4llm

        result = pymupdf4llm.to_markdown(str(pdf_path))
        text = (
            "\n\n".join(str(r) for r in result) if isinstance(result, list) else result
        )
        return _post_process_pdf_text(text)
    except Exception as e:
        logger.error(f"pymupdf4llm failed conversion of {Path(pdf_path).name}: {e}")
        return ""


def _post_process_pdf_text(text: str) -> str:
    """Fix common text-extraction artefacts produced by pymupdf4llm.

    Applied automatically by pdf_to_text() — you rarely need to call this directly.

    Fixes applied (in order):
    1. Corrupted table rows containing only pipes and whitespace — removed.
    2. Spurious ``- `` bullet prefixes inside Mermaid diagram fences — removed.
    3. Missing spaces at digit–lowercase-letter boundaries (e.g. ``250bar`` → ``250 bar``).
    4. Missing spaces at digit–Title-case boundaries (e.g. ``3Phase`` → ``3 Phase``).
    5. Missing spaces at camelCase boundaries (e.g. ``hydraulicPump`` → ``hydraulic Pump``).
    6. All-lowercase compound words segmented via wordninja (optional dependency).
    7. Runs of 3+ blank lines collapsed to 2.

    Parameters
    ----------
    text : str
        Raw Markdown string from pymupdf4llm.

    Returns
    -------
    str
        Cleaned text string.
    """
    # 1. Remove corrupted / empty table rows (lines that are only pipes and whitespace)
    text = re.sub(r"^\|[\s|]*\|\s*$", "", text, flags=re.MULTILINE)

    # 2. Strip "- " bullets inside Mermaid code fences
    def _fix_mermaid(m: re.Match) -> str:
        return (
            m.group(1) + re.sub(r"^- ", "", m.group(2), flags=re.MULTILINE) + m.group(3)
        )

    text = re.sub(r"(```mermaid\n)(.*?)(```)", _fix_mermaid, text, flags=re.DOTALL)

    # 3. Digit–lowercase-letter boundaries
    text = re.sub(r"([a-z])(\d)", r"\1 \2", text)
    text = re.sub(r"(\d)([a-z])", r"\1 \2", text)

    # 4. Digit before a Title-case word
    text = re.sub(r"(\d)([A-Z][a-z])", r"\1 \2", text)

    # 5. camelCase split
    text = re.sub(r"([a-z])([A-Z])", r"\1 \2", text)

    # 6. All-lowercase compound words via wordninja (graceful fallback if not installed)
    try:
        import wordninja

        def _segment_word(m: re.Match) -> str:
            word = m.group()
            if word.isupper():          # skip acronyms like "HVAC"
                return word
            parts = wordninja.split(word)
            if len(parts) <= 1:
                return word
            if word[0].isupper() and parts[0]:
                parts[0] = parts[0].capitalize()
            return " ".join(parts)

        text = re.sub(r"\b[A-Za-z]{6,}\b", _segment_word, text)
    except ImportError:
        logger.debug("wordninja not installed; skipping compound-word segmentation")

    # 7. Collapse extra blank lines left by removed rows
    text = re.sub(r"\n{3,}", "\n\n", text)

    return text.strip()
```

---

## Utilizzo completo — esempio end-to-end

```python
from rag_utils import create_in_memory_rag
from doc_utils import load_context_files
from openai import AzureOpenAI
import os
from dotenv import load_dotenv

load_dotenv()

# ── 1. Carica tutti i documenti dalla directory ──────────────────────────────
raw_text = load_context_files("docs/")      # directory con .docx e/o .pdf

# ── 2. Costruisci il vector store (una volta sola per sessione) ──────────────
vector_db = create_in_memory_rag(raw_text or "")

# ── 3. Per ogni query, recupera i chunk più rilevanti ────────────────────────
query = "Come si diagnostica un'anomalia alla pompa idraulica?"

docs = vector_db.similarity_search(query, k=5)

retrieved_context = "\n\n".join(
    f"=== SEZIONE DOCUMENTAZIONE {i + 1} ===\n{doc.page_content}"
    for i, doc in enumerate(docs)
)

# ── 4. Chiama l'LLM con il contesto recuperato ───────────────────────────────
client = AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_API_KEY") or "",
    api_version=os.getenv("AZURE_OPENAI_API_VERSION") or "",
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT") or "",
)

system_prompt = "Sei un esperto di manutenzione industriale. Usa solo le informazioni fornite nel contesto."

user_prompt = f"""
Contesto dalla documentazione tecnica:
{retrieved_context}

Domanda: {query}
"""

response = client.chat.completions.create(
    model=os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME") or "",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user",   "content": user_prompt},
    ],
    temperature=0,
    max_completion_tokens=500,
)

print(response.choices[0].message.content)
```

---

## Parametri chiave e quando cambiarli

| Parametro | Default | Quando cambiarlo |
|---|---|---|
| `chunk_size` | 2000 caratteri | Riduci (es. 800) per documenti tecnici/tabulari; aumenta (es. 3000) per testi narrativi |
| `chunk_overlap` | 400 caratteri | ~20% del chunk_size è un buon punto di partenza |
| `k` in `similarity_search` | 5 | Aumenta se le risposte sembrano incomplete; riduci se superi il context window |
| Modello embedding | `BAAI/bge-small-en-v1.5` | Usa `BAAI/bge-m3` per documenti in italiano o multilingua |

---

## InMemoryVectorStore vs persistenza su disco

`InMemoryVectorStore` vive **solo in RAM**: va ricostruito ad ogni avvio. Per documenti statici il rebuild è rapido (< 1s per ~50 pagine). Se hai molti documenti o vuoi persistenza, sostituisci con Chroma:

```python
# pip install chromadb langchain-community
from langchain_community.vectorstores import Chroma

# Prima esecuzione: costruisce e salva
vector_db = Chroma.from_texts(chunks, embedding_model, persist_directory="./chroma_db")

# Esecuzioni successive: ricarica senza rembedding
vector_db = Chroma(persist_directory="./chroma_db", embedding_function=embedding_model)
```
