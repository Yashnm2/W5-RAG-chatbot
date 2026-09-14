# W5 RAG — complete run guide

This guide is for the terminal application in [`[Lodge_Lab] Starter.py`](../%5BLodge_Lab%5D%20Starter.py). It follows the code that ships in this repository, not a generic RAG tutorial. Work through the first-run checklist once; use the function reference and troubleshooting sections when you change data or configuration.

> **Before you start:** This project contacts paid or metered external services. A file you ingest is sent to OpenAI for embeddings and stored as text metadata in Pinecone. The optional website command also sends the requested URL to Jina Reader. Keep secrets and material you cannot share out of the project.

## Contents

1. [Requirements and accounts](#requirements-and-accounts)
2. [Install on macOS or Linux](#install-on-macos-or-linux)
3. [Install on Windows](#install-on-windows)
4. [Configure `.env`](#configure-env)
5. [First run, from empty index to answer](#first-run-from-empty-index-to-answer)
6. [Operating the chatbot](#operating-the-chatbot)
7. [Configuration reference](#configuration-reference)
8. [Function reference](#function-reference)
9. [Troubleshooting](#troubleshooting)
10. [Costs, data, and limits](#costs-data-and-limits)
11. [Official sources](#official-sources)

## Requirements and accounts

- **Python 3.9 or later**, with `venv` and `pip`. Python 3.10–3.13 is a sensible choice for this workshop: Pinecone's Python SDK documentation says it requires 3.9+ and has been tested through 3.13. The original README claimed 3.8, which is too old for that SDK requirement.
- An **OpenAI API key** with API billing/credits available for embeddings and chat requests. A ChatGPT subscription is separate from API billing.
- A **Pinecone API key** and permission to create or access a serverless index. The default first-run location is AWS `us-east-1`; check your plan's available regions in Pinecone before changing it.
- Network access to OpenAI and Pinecone. Website ingestion additionally needs access to Jina Reader and the target page.
- A terminal opened in this repository's root directory. Paths in the program are relative to the current working directory.

You do **not** need FastAPI, Uvicorn, a browser app, or a running web server. The program is interactive in the terminal.

## Install on macOS or Linux

From the project directory:

```bash
python3 --version
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
cp .env.example .env
```

Check that your prompt shows `(.venv)` or run `which python` to confirm the environment. Python's `venv` isolates this project's packages; `.venv` is intentionally excluded from Git. Edit `.env` in your editor and fill in the two keys. Do not paste keys into terminal commands that may be saved in shell history.

To start the application:

```bash
python "[Lodge_Lab] Starter.py"
```

The quotes around the filename matter because it contains a space and brackets. On a successful start you will see import and configuration messages followed by `You:`. The script creates the configured Pinecone index if it does not exist, then waits ten seconds. Index creation is a real remote write and may take longer than ten seconds to become queryable.

## Install on Windows

In **PowerShell**, from the project directory:

```powershell
py -3 --version
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Edit `.env`, then run:

```powershell
python '.\[Lodge_Lab] Starter.py'
```

In **Command Prompt**, activate with `.venv\Scripts\activate.bat`, copy the template with `copy .env.example .env`, and use `python "[Lodge_Lab] Starter.py"`.

If PowerShell blocks the activation script, Python's own `venv` documentation gives `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` as an option. You can avoid changing execution policy by running `.\.venv\Scripts\python.exe` directly for the install and launch commands instead.

## Configure `.env`

Copy `.env.example` and replace the placeholder values for **both required keys**. The script calls `load_dotenv()` at import time, so it reads `.env` in the working directory. Do not use the keys from the local workshop copy or put real keys in `.env.example`.

```dotenv
OPENAI_API_KEY=your_openai_api_key_here
PINECONE_API_KEY=your_pinecone_api_key_here
PINECONE_INDEX=my-first-rag
PINECONE_CLOUD=aws
PINECONE_REGION=us-east-1
PINECONE_NAMESPACE=__default__
EMBED_MODEL=text-embedding-3-small
EMBED_DIMENSIONS=1024
CHAT_MODEL=gpt-4o-mini
```

The first run checks only that the two key variables are nonempty; it does **not** validate that placeholders have been replaced. If you leave placeholders, the first provider request will fail authentication. Create keys in the provider consoles and store them only in `.env` or environment variables. `.gitignore` excludes `.env`, but check `git status` before publishing any changes.

For a first run, keep the other defaults. The configured `text-embedding-3-small` request asks OpenAI for 1,024-dimensional vectors, and the script creates a 1,024-dimensional Pinecone index. The same model and dimension must be used for both documents and questions.

## First run, from empty index to answer

1. **Prepare a tiny source.** Create the ingestion directory and copy the included example:

   ```bash
   mkdir -p "Files to insert (PDF or TXT)"
   cp examples/sample-knowledge.txt "Files to insert (PDF or TXT)/"
   ```

   On Windows PowerShell, use:

   ```powershell
   New-Item -ItemType Directory -Force 'Files to insert (PDF or TXT)'
   Copy-Item examples\sample-knowledge.txt 'Files to insert (PDF or TXT)\'
   ```

2. **Start the program** with the OS-specific command above. It prints the index, embedding model, and dimension; verify they are the values you expect. The first run may create the index. A later run reconnects to the existing index.
3. At `You:`, type exactly `ingest_files()` and press Enter. Expect one file processed and one or more chunks stored. Ingestion makes one OpenAI embedding call and one Pinecone upsert per chunk; a long document creates many calls.
4. At the next `You:`, ask `When is the workshop library open?` The script embeds the question, retrieves up to three chunks, and asks the chat model to answer using that context. An answer should mention Monday–Friday and 09:00–18:00. If a just-upserted record is not yet returned, wait briefly and ask again; Pinecone documents eventual consistency for new or changed records.
5. Ask `What is the library's phone number?` The supplied file has no phone number. The prompt tells the model to say it lacks the information. This is a useful grounding check, though the application does not enforce the instruction mechanically.
6. Type `quit` or `exit` to leave. The vectors stay in Pinecone after the terminal exits.

Move the example out of the ingestion folder after the test. The script does **not** move or delete processed files; it reprocesses every supported file each time you run `ingest_files()`.

## Operating the chatbot

The terminal accepts a question or one of these **exact** commands:

| Input | What it does | Caution |
| --- | --- | --- |
| Any other nonempty text | Sends a retrieval query and chat request. | Incurs provider usage. The answer is not a verified citation. |
| `ingest_files()` | Reads all `.pdf`, `.txt`, and `.md` files directly inside `Files to insert (PDF or TXT)`. | Repeating it re-embeds and upserts; it does not recurse into subfolders. |
| `scrape_website()` | Prompts for an `http://` or `https://` URL and ingests the extracted text. | Sends the URL to Jina Reader; use a public page you are allowed to process. |
| `clearDB()` | Asks you to type `yes`, then deletes vectors in the configured namespace. | Destructive. Other namespaces are unaffected. |
| `quit`, `exit`, or `bye` | Stops the local chat loop. | Does not delete data or the index. |

### Ingest your own documents

Place supported files in `Files to insert (PDF or TXT)` and run `ingest_files()`. TXT and Markdown are read as UTF-8. PDF text is extracted with pypdf; image-only scans need OCR first. The script chunks extracted text by **characters** (`800` per chunk, `100` characters overlap), not tokens or sentences. It stores the original text as Pinecone metadata, so avoid confidential documents unless your policies allow this transfer.

Each chunk gets an ID of `<filename-without-extension>_<chunk-number>`. Re-ingesting a file with the same stem **overwrites** matching IDs, but does not remove any older trailing IDs if the new file yields fewer chunks. Two files with the same stem, such as `notes.pdf` and `notes.txt`, collide. Use distinct filenames or a fresh namespace for experiments. The script reports a file as processed even if extraction yielded no chunks; check the `Total chunks stored` line as well.

### Ingest a website

At `You:`, type `scrape_website()`, then a public URL such as `https://example.com`. The code prefixes it with `https://r.jina.ai/`, asks Jina Reader for text, chunks the response, and stores the chunks. Jina Reader output quality depends on the page, its access controls, and its rendering. This code sets a 30-second request timeout. A network exception during this command is not caught inside the command branch, so it can terminate the program; restart and try a reachable page if that happens.

Website chunk IDs use the site's domain plus a number. Ingesting multiple pages from the same domain can overwrite previous chunks. This is an intentional simplification of the starter, not a page-level archive.

### Ask questions and interpret answers

The query path embeds your question, queries the current Pinecone namespace with `top_k=3`, concatenates the returned text, and calls the configured chat model. The prompt asks the model to admit when the context does not contain an answer, but the output is generated text and may still be wrong. There is no source URL, document name, score, or citation in the terminal answer. Verify important facts against source files. If the namespace is empty, the model receives empty context.

### Clear data safely

`clearDB()` deletes all vectors in `PINECONE_NAMESPACE` after you type `yes`. Pinecone documents `delete_all=True` as a **namespace-scoped** operation; it does not remove the index itself. It is not an undoable action. Verify `PINECONE_INDEX` and `PINECONE_NAMESPACE` in the startup output before confirming. A namespace is a useful way to isolate workshop experiments.

## Configuration reference

| Variable | Read by this script? | Effect and safe change |
| --- | --- | --- |
| `OPENAI_API_KEY` | Yes; required | Authenticates both embedding and chat calls. Use your own API key. |
| `PINECONE_API_KEY` | Yes; required | Authenticates index management and data operations. |
| `PINECONE_INDEX` | Yes | Index name. The script creates it if missing. Existing indexes are reused without checking their dimension or metric. |
| `PINECONE_CLOUD` | Yes | Used only in `ServerlessSpec` for a **new** index; defaults to `aws`. Does not relocate an existing index. |
| `PINECONE_REGION` | Yes | Used only for a **new** index; defaults to `us-east-1`. Available regions depend on Pinecone plan. |
| `PINECONE_NAMESPACE` | Yes | Passed to upsert, query, and delete; defaults to `__default__`, Pinecone's explicit default namespace name. |
| `EMBED_MODEL` | Yes | OpenAI embedding model. `dimensions` must be supported by the selected model. Changing models can make old and new vectors incompatible even if dimensions match semantically. |
| `EMBED_DIMENSIONS` | Yes | Integer output vector size, default `1024`. Must match the Pinecone index dimension. Changing it for an existing index will not resize that index. |
| `CHAT_MODEL` | Yes | Model for answers, default `gpt-4o-mini`. The model must support the Chat Completions parameters used here. |

The index uses cosine similarity. Pinecone's guidance for external embeddings says the index dimension and metric should suit the embedding model. If you want a different dimension, use a **new index name** with that dimension, then re-ingest your documents. If you want to test a different document set without changing dimension, use a **new namespace**. Do not mix embedding models in the same namespace merely because they emit the same number of values.

## Function reference

The script defines **twelve functions**. This section covers every one, in source order. Importing the file has immediate side effects: it loads `.env`, validates keys, creates OpenAI/Pinecone clients, and may create a Pinecone index. For that reason, do not import it just to call `chunk_text` without controlling configuration or mocking these clients.

### `create_embedding(text)`

- **Input:** A nonempty string.
- **Returns:** A list of floating-point numbers, requested at `EMBED_DIMENSIONS` length.
- **How:** Calls `openai_client.embeddings.create(model=EMBED_MODEL, input=text, dimensions=EMBED_DIMENSIONS)` and returns the first embedding.
- **Used by:** `store_in_pinecone` for document chunks and `retrieve_from_pinecone` for questions.
- **Effects/failures:** Makes a billable OpenAI API request. Empty or oversized input, invalid key/model/dimension, quota, or network errors propagate to the caller. The function does not batch multiple chunks.

### `store_in_pinecone(text, source_name, chunk_id)`

- **Inputs:** Passage text, source label, and a chunk number.
- **Returns:** Nothing; prints a stored message.
- **How:** Embeds `text`, makes ID `f"{source_name}_{chunk_id}"`, and upserts one vector with metadata `{"text": text, "source": source_name}` into `PINECONE_NAMESPACE`.
- **Effects/failures:** Makes one OpenAI request and one Pinecone write per call. Reusing an ID replaces that vector. Exceptions propagate, so an ingest may stop partway through or report a file failure. Text is stored remotely as metadata.

### `read_file(file_path)`

- **Input:** Path to a local file.
- **Returns:** Extracted text, or `""` if the file is missing or reading fails.
- **How:** Uses `PdfReader` for `.pdf` (joins page text with newlines), otherwise opens the path as UTF-8 text. The folder command filters to PDF/TXT/MD before calling it.
- **Effects/failures:** Reads local data, prints progress/errors, and catches file/PDF exceptions. It cannot OCR scanned pages. Pages with no extractable text contribute a newline. A PDF with no usable text leads to zero chunks.

### `store_file_in_pinecone(file_path, chunk_size=800, overlap=100)`

- **Inputs:** File path and optional character chunk size/overlap.
- **Returns:** Number of chunks upserted; returns `0` when extraction yields no text.
- **How:** Calls `read_file`, derives `source_name` from the filename stem, calls `chunk_text`, then calls `store_in_pinecone` for every chunk starting at index `0`.
- **Effects/failures:** Potentially many remote writes and embedding calls. A failure during the loop does not roll back earlier chunks. Repeating with a shorter file may leave old higher-numbered chunk IDs in Pinecone.

### `retrieve_from_pinecone(query, top_k=3)`

- **Inputs:** Question/search text and maximum result count.
- **Returns:** A list of the matched passages' `metadata["text"]` values, in Pinecone's returned order.
- **How:** Embeds the query and calls `pinecone_index.query(vector=..., top_k=..., include_metadata=True, namespace=...)`.
- **Effects/failures:** Makes one OpenAI embedding request and one Pinecone read. Assumes every match has text metadata; manually inserted records without it can raise an error. It does not return scores or source labels to the caller.

### `clear_database()`

- **Inputs:** None.
- **Returns:** `True` if Pinecone accepted the delete, `False` after an exception.
- **How:** Calls `pinecone_index.delete(delete_all=True, namespace=PINECONE_NAMESPACE)`.
- **Effects/failures:** Permanently deletes records in that namespace. It catches and prints errors. The function itself has **no confirmation**; confirmation exists only in the `run_chatbot` command path. Do not call it directly unless you intend to delete those records.

### `ingest_files_from_folder(folder_path="Files to insert (PDF or TXT)")`

- **Input:** Folder path; the default is relative to where the process is launched.
- **Returns:** Number of files put in the `processed_files` list, not necessarily the number with nonzero chunks.
- **How:** Lists the folder's immediate files ending in `.pdf`, `.txt`, or `.md`, calls `store_file_in_pinecone` for each, and prints a summary.
- **Effects/failures:** May make many remote calls. It catches exceptions per file and continues to the next one. It does not clear old vectors, remove local files, recurse into subfolders, or prevent repeated ingestion.

### `scrape_website(url)`

- **Input:** A URL string. `run_chatbot` requires it to start with `http://` or `https://`; the function itself does not validate it.
- **Returns:** Jina Reader's text when HTTP status is `200`, otherwise `""`.
- **How:** Performs `requests.get(f"https://r.jina.ai/{url}", timeout=30)` and returns the response body.
- **Effects/failures:** Sends the URL to Jina Reader; this function alone does **not** write Pinecone. Non-200 responses are reported. Request exceptions propagate because there is no `try`/`except` inside this function.

### `chunk_text(text, chunk_size=800, overlap=100)`

- **Inputs:** Text and two character counts; requires `chunk_size > 0` and `0 <= overlap < chunk_size`.
- **Returns:** A list of non-whitespace chunks.
- **How:** Takes `text[start:start + chunk_size]`, then advances by `chunk_size - overlap` characters. This preserves overlap to help retrieval at boundaries.
- **Effects/failures:** Local computation only. Invalid size/overlap raises `ValueError`. It does not preserve sentence boundaries or count model tokens; large unusual characters or source formatting can still affect quality.

### `get_relevant_context(question, top_k=3)`

- **Inputs:** Question and result limit.
- **Returns:** A single string with retrieved chunks joined by two newlines.
- **How:** Calls `retrieve_from_pinecone` and joins its text values.
- **Effects/failures:** Has the retrieval API effects noted above. Empty results produce an empty string. It discards source IDs and scores.

### `chat_with_rag(user_question)`

- **Input:** User's question.
- **Returns:** The text content of the first chat completion choice; could be `None` if a provider response has no content.
- **How:** Gets up to three chunks, places them in a prompt that asks for context-grounded answers and an explicit no-information response, then calls `openai_client.chat.completions.create(model=CHAT_MODEL, temperature=0.7, max_tokens=500)`.
- **Effects/failures:** One embedding request, one Pinecone query, and one chat completion per question. Provider failures propagate to `run_chatbot`, which prints an error. It does not validate source accuracy or return citations. Retrieved source text is sent to OpenAI in the chat request.

### `run_chatbot()`

- **Inputs/returns:** No arguments; loops until `quit`, `exit`, or `bye`, then returns `None`.
- **How:** Prints instructions, reads `input("You: ")`, routes the three exact special commands, and sends ordinary nonempty text to `chat_with_rag`.
- **Effects/failures:** File ingestion and website ingestion cause remote writes; `clearDB()` requires `yes` before calling `clear_database`. Ordinary question errors are caught and printed. EOF (`Ctrl-D` / redirected input ending), an interrupt, and some exceptions in special-command branches are not caught, so they can exit the process. Exiting does not remove stored vectors.

The bottom `if __name__ == "__main__":` block is the launch point. It calls `run_chatbot()` when you run the file directly. The original workshop copy had this block inside a triple-quoted string; this repository enables it so the documented command opens the chat.

## Troubleshooting

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| `ModuleNotFoundError` | Dependencies installed into another Python interpreter. | Activate `.venv`; run `python -m pip install -r requirements.txt` and then launch with that `python`. |
| `PINECONE_API_KEY not found` / `OPENAI_API_KEY not found` | `.env` missing or launched from another folder. | Copy `.env.example` to `.env`, fill it, and run from the repo root. Do not print your keys to debug. |
| Authentication or quota error | Placeholder/expired key, wrong project permissions, or exhausted provider credits. | Verify the key and billing status in the provider console. Restart after changing `.env`. |
| Pinecone dimension mismatch | Existing index dimension differs from `EMBED_DIMENSIONS`, or you changed the embedding model/dimension. | Inspect index settings in Pinecone. Use matching dimensions or a new index and re-ingest. |
| Index not ready immediately after creation | The fixed ten-second startup wait was insufficient. | Wait and restart. The script will find the existing index on its next launch. |
| `ingest_files()` finds nothing | Missing folder, unsupported extension, wrong current directory, or files only in subfolders. | Make the folder at repo root; put PDF/TXT/MD files directly inside it. |
| PDF reports zero useful text | Image-only scan or extraction failure. | Use a searchable PDF or OCR it before ingestion. pypdf does not perform OCR. |
| Answer says no information after ingestion | Empty namespace, ingestion failure, eventual consistency delay, or irrelevant top-three chunks. | Check `Total chunks stored`, index and namespace, wait briefly, and ask a more specific question. |
| Website command fails or exits | Target page inaccessible, Jina Reader error, or 30-second timeout. | Try a public, reachable page; restart the program after an uncaught request exception. |
| Wrong or stale answer | Old chunk IDs, duplicate stems, broad retrieval, or model error. | Use a fresh namespace or carefully clear the intended namespace and re-ingest; compare with the source. |
| PowerShell cannot activate `.venv` | Script execution policy. | Run `.venv\Scripts\python.exe` directly or use the `CurrentUser` execution-policy option in Python's documentation. |

## Costs, data, and limits

- **OpenAI:** Each stored chunk uses an embedding request. Each question uses another embedding request and a chat completion. Pricing and model availability change; check the linked official model/pricing pages before large ingests.
- **Pinecone:** The app may create a serverless index; storage, reads, and writes can have plan limits or charges. It stores passage text as metadata, not only vectors. It can take a short time for writes to appear in queries.
- **Jina Reader:** Website ingestion asks a third-party service to fetch the URL. The code does not attach a Jina API key; service policies or access may change. Review its current documentation before relying on it for a workshop.
- **Local secrets:** `.env` and the ingestion folder are ignored. That prevents ordinary new Git adds, but does not protect secrets previously committed elsewhere or files you explicitly force-add. Use a private repository for sensitive experiments, and never commit real keys or private source documents.
- **Scale:** The script upserts one chunk at a time and uses fixed character chunks with no deduplication, retries, batch processing, citations, or automatic cleanup. Start with the sample, then a small document. For larger workloads, improve the pipeline rather than treating this as production ingestion.

## Official sources

The run instructions and limitations above were checked against the code in this repository and these primary sources (reviewed September 2026):

- [Python `venv` documentation](https://docs.python.org/3/library/venv.html) — environment creation, activation, and PowerShell policy guidance.
- [Python virtual environments and `pip`](https://docs.python.org/3/tutorial/venv.html) — installing from `requirements.txt`.
- [Pinecone Python SDK overview](https://docs.pinecone.io/reference/sdks/python/overview) — supported Python versions and SDK install.
- [Pinecone: create an index](https://docs.pinecone.io/guides/index-data/create-an-index) — external vector dimensions, metric, cloud, and region.
- [Pinecone: delete records](https://docs.pinecone.io/guides/manage-data/delete-data) — namespace-scoped `delete_all` and eventual consistency.
- [Pinecone: manage namespaces](https://docs.pinecone.io/guides/manage-data/manage-namespaces) — explicit `__default__` namespace.
- [OpenAI: create embeddings](https://developers.openai.com/api/reference/python/resources/embeddings/methods/create) — `dimensions` support on `text-embedding-3` models and input limits.
- [OpenAI: `text-embedding-3-small`](https://developers.openai.com/api/docs/models/text-embedding-3-small) and [`gpt-4o-mini`](https://developers.openai.com/api/docs/models/gpt-4o-mini) — the defaults used here; see the linked model pages for current pricing and availability.
- [pypdf text extraction](https://pypdf.readthedocs.io/en/latest/user/extract-text.html) — searchable PDFs versus scans and OCR limits.
- [Jina Reader project documentation](https://github.com/jina-ai/reader) — the `https://r.jina.ai/<URL>` reading route.
