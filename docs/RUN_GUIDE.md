<!-- markdownlint-disable MD013 MD033 -->

# W5 RAG chatbot — complete beginner run guide


This project is a **terminal chatbot**, not a website. It reads your PDF, TXT, or Markdown files, sends their text to APIYI to create embeddings, stores the text and embeddings in Pinecone, and uses the retrieved text to answer questions.


## What you need

- A Mac or Windows computer with internet access.
- Access to this GitHub repository. It is private, so GitHub must show the repository while you are signed in.
- Python 3.10 or newer. Python 3.12–3.14 is recommended.
- An APIYI API key with available credit.
- A Pinecone account and API key. The free Starter plan is enough for this workshop if its limits are not already used.


## Setup map

Complete these steps in order:

1. [Download the project](#1-download-the-project)
2. [Install Python](#2-install-python)
3. [Create the APIYI key](#3-create-the-apiyi-key)
4. [Create the Pinecone key](#4-create-the-pinecone-key)
5. [Open a terminal in the project folder](#5-open-a-terminal-in-the-project-folder)
6. [Create the Python environment](#6-create-the-python-environment)
7. [Create and edit `.env`](#7-create-and-edit-env)
8. [Run the chatbot](#8-run-the-chatbot)
9. [Test with the included file](#9-test-with-the-included-file)

Do not skip a step. Keep both API keys ready, but never paste them into chat, screenshots, or GitHub.

## 1. Download the project

### Recommended: Download ZIP

The browser steps are the same on Mac and Windows:

1. Sign in to GitHub.
2. Open [W5-RAG-chatbot](https://github.com/Yashnm2/W5-RAG-chatbot).
3. Click the green **Code** button.
4. Click **Download ZIP**.
5. Open the downloaded ZIP file to extract it.
6. Find the extracted `W5-RAG-chatbot-main` folder. Move it somewhere easy to find, such as Documents.


### Alternative: Git command


```text
git clone https://github.com/Yashnm2/W5-RAG-chatbot.git
cd W5-RAG-chatbot
```

The rest of this guide calls the extracted or cloned folder the **project folder**.


## 2. Create the APIYI key

APIYI supplies both the embedding model and the chat model used by this project.

1. Open the [APIYI website](https://api.apiyi.com/) and create an account.
2. Verify your email and sign in.
3. Open the [APIYI token page](https://api.apiyi.com/token).
4. Copy the default token, or click **New**, name it `w5-rag-chatbot`, and confirm.
5. Save the token temporarily in a private place. APIYI keys normally start with `sk-`.
6. In the APIYI console, check that the account has trial credit or a positive balance. Add credit if required.

Call this value your `APIYI_API_KEY`. Do not include quotation marks when it is placed in `.env` later.

## 3. Create the Pinecone key

Pinecone stores and searches the document chunks.

1. Open the [Pinecone console](https://app.pinecone.io/) and create or sign in to an account.
2. Create or select a project. A default project is fine.
3. In that project, open **API keys**.
4. Click **Create API key**.
5. Name it `w5-rag-chatbot`.
6. Choose **All** permissions if that is the only option on the Starter or Builder plan. The key must be able to create an index and read, write, and delete records.
7. Click **Create key**.
8. Copy the key immediately and keep it private. Pinecone does not show the full key again after the dialog closes.

Call this value your `PINECONE_API_KEY`.

### Do I create a Pinecone index manually?

**No.** On the first launch, the Python script automatically creates this serverless index if it does not already exist:

| Setting | Default value |
| --- | --- |
| Index name | `my-first-rag` |
| Vector type | Dense |
| Dimension | `1024` |
| Metric | Cosine |
| Cloud | AWS |
| Region | `us-east-1` |
| Namespace | `__default__` |

`aws` and `us-east-1` are intentionally used because Pinecone Starter and Builder plans support that region. Do not manually create an index with the same name but different settings.

## 4. Open a terminal in the project folder

All remaining commands must be run inside the folder containing these files:

```text
[Lodge_Lab] Starter.py
requirements.txt
.env.example
docs
examples
```

| macOS | Windows |
| --- | --- |
| 1. Open **Terminal** using Spotlight.<br>2. Type `cd`, then press the Space bar once.<br>3. Drag the project folder from Finder into Terminal.<br>4. Press Return. | 1. Open the project folder in File Explorer.<br>2. Click the address bar, type `powershell`, and press Enter.<br>3. A PowerShell window opens in that folder. |

Confirm that you are in the correct place:

| macOS Terminal | Windows PowerShell |
| --- | --- |
| `ls` | `Get-ChildItem` |

You must see `[Lodge_Lab] Starter.py` and `requirements.txt`. If not, stop and reopen the terminal in the correct folder. A common mistake is stopping in the Downloads folder or in the outer ZIP folder.

## 5. Create the Python environment

Run **one line at a time**. Wait for each command to finish before running the next one.

| Step | macOS Terminal | Windows PowerShell |
| --- | --- | --- |
| Create an isolated environment | `python3 -m venv .venv` | `py -3 -m venv .venv` |
| Upgrade its installer | `.venv/bin/python -m pip install --upgrade pip` | `.\.venv\Scripts\python.exe -m pip install --upgrade pip` |
| Install this project's packages | `.venv/bin/python -m pip install -r requirements.txt` | `.\.venv\Scripts\python.exe -m pip install -r requirements.txt` |

The last command may take a few minutes. Warnings about a newer `pip` version are harmless. A red `ERROR` or `ModuleNotFoundError` is not harmless; use [Troubleshooting](#troubleshooting).

These commands deliberately use the environment's Python directly. You do not need to activate the environment, so Windows PowerShell execution-policy settings cannot block this guide.

## 6. Create and edit `.env`

The `.env` file holds the two private keys and the safe defaults used by the script. If a `.env` file already exists and contains your keys, skip Step 7.1 so you do not overwrite it.

### 6.1 Create it from the template

| macOS Terminal | Windows PowerShell |
| --- | --- |
| `cp .env.example .env` | `Copy-Item .env.example .env` |

### 6.2 Open it

| macOS Terminal | Windows PowerShell |
| --- | --- |
| `open -e .env` | `notepad .env` |

Replace only the two placeholder values. The finished file should have this shape:

```dotenv
APIYI_API_KEY=sk-your-real-apiyi-key
PINECONE_API_KEY=your-real-pinecone-key

PINECONE_INDEX=my-first-rag
PINECONE_CLOUD=aws
PINECONE_REGION=us-east-1
PINECONE_NAMESPACE=__default__

EMBED_MODEL=text-embedding-3-small
EMBED_DIMENSIONS=1024
CHAT_MODEL=gpt-4o-mini
```

Important checks before saving:


Close the editor after saving. The repository's `.gitignore` excludes `.env`, but you must still treat it as a secret.

## 7. Run the chatbot

Use the same terminal, still inside the project folder:

| macOS Terminal | Windows PowerShell |
| --- | --- |
| `.venv/bin/python "[Lodge_Lab] Starter.py"` | `.\.venv\Scripts\python.exe '.\[Lodge_Lab] Starter.py'` |

On the first run, the script contacts Pinecone and may create `my-first-rag`. It then waits about ten seconds. A successful launch ends with output similar to:

```text
[SUCCESS] All libraries imported successfully!
[SUCCESS] Environment variables loaded!
[SUCCESS] Connected to Pinecone index: my-first-rag
[COMPLETE] SETUP COMPLETE! Ready to build your RAG chatbot!
You:
```

The exact surrounding lines may differ. The important part is the final `You:` prompt.

If the script reports that the index is still initializing, wait 30 seconds and run the same launch command again. Do not create a second index.

## 8. Test with the included file

First type `exit` at `You:` if the chatbot is currently running. Then prepare the sample file:

| macOS Terminal | Windows PowerShell |
| --- | --- |
| `mkdir -p "Files to insert (PDF or TXT)"`<br>`cp examples/sample-knowledge.txt "Files to insert (PDF or TXT)/"` | `New-Item -ItemType Directory -Force 'Files to insert (PDF or TXT)'`<br>`Copy-Item examples\sample-knowledge.txt 'Files to insert (PDF or TXT)\'` |

Launch the chatbot again using the command from Step 8. At `You:`, type exactly:

```text
ingest_files()
```

Wait until the summary says:

```text
Files processed: 1
Total chunks stored: 1
```

At the next `You:` prompt, ask:

```text
When is the workshop library open?
```

The answer should say that it is open Monday to Friday from 09:00 to 18:00. Wording can differ.

Now ask this grounding test:

```text
What is the library's phone number?
```

The sample file has no phone number, so the bot should say it does not have that information.

Type `exit` to close the chatbot. The Pinecone data remains stored.

## Use your own documents

1. Exit the chatbot.
2. Open `Files to insert (PDF or TXT)` inside the project folder.
3. Remove `sample-knowledge.txt` if you do not want it mixed with your data.
4. Copy your `.pdf`, `.txt`, or `.md` files directly into the folder. Subfolders are not scanned.
5. Launch the chatbot again.
6. Type `ingest_files()`.
7. Check that `Total chunks stored` is greater than zero.
8. Ask a specific question whose answer appears in the document.

Use searchable PDFs. A scanned PDF made only of images needs OCR first; this project does not perform OCR.

Do not repeatedly run `ingest_files()` on the same folder unless you intend to reprocess it. Files with the same name before the extension, such as `notes.pdf` and `notes.txt`, overwrite one another's chunk IDs. If a replacement file becomes shorter, older trailing chunks can remain; use a new namespace or clear the current one before a clean re-ingest.

## Use a public webpage

At `You:`, type:

```text
scrape_website()
```

Paste a complete public URL beginning with `https://`, then press Enter. The script sends the URL to Jina Reader, chunks the returned text, and stores it in Pinecone.

Pages requiring a login, blocking automated access, or heavily depending on browser interaction may fail. Ingesting two pages from the same domain can overwrite earlier chunks because this starter uses the domain in its vector IDs.

## Chatbot commands

| What you type | Result |
| --- | --- |
| A normal question | Retrieves up to three chunks and generates an answer. |
| `ingest_files()` | Loads supported files from the ingestion folder. |
| `scrape_website()` | Asks for and loads one public webpage. |
| `clearDB()` | Offers to delete every vector in the configured namespace. |
| `exit`, `quit`, or `bye` | Stops the program without deleting Pinecone data. |

Commands are case-sensitive except for the three exit words. Type the parentheses in `ingest_files()`, `scrape_website()`, and `clearDB()`.

## Pinecone data and safe cleanup

The project uses `PINECONE_NAMESPACE=__default__`. A namespace separates groups of vectors inside one index.

To remove this project's stored vectors from the terminal:

1. Check the startup lines for the intended index and namespace.
2. Type `clearDB()` at `You:`.
3. Type `yes` only if you want to permanently delete every vector in that namespace.

This does not delete the Pinecone index. There is no undo. If an index is shared with someone else, use a unique namespace instead of clearing shared data, for example:

```dotenv
PINECONE_NAMESPACE=yash-workshop
```

After changing a namespace, restart the chatbot and ingest the documents again.

If you change `EMBED_MODEL` or `EMBED_DIMENSIONS`, use a new `PINECONE_INDEX` name and re-ingest everything. Pinecone index names must be lowercase, use only letters, numbers, and hyphens, and be no longer than 45 characters.

## Starting again on another day

You do not need to reinstall anything.

1. Open a terminal in the project folder using Step 5.
2. Run the Step 8 launch command for your operating system.
3. Ask questions immediately if the data is already in Pinecone, or ingest new files first.

## Troubleshooting

Match the first useful error message, not the final generic message.

| Problem or message | Fix |
| --- | --- |
| GitHub shows `404` | Sign in with an account that has access, or ask the owner to grant it. |
| `python3: command not found` on Mac | Install Python from python.org, close Terminal, reopen it, and retry. |
| `py is not recognized` on Windows | Try `python --version`. If it works, replace `py -3` with `python`. Otherwise reinstall Python with its PATH option. |
| `No such file`, `cannot find path`, or `requirements.txt` missing | The terminal is in the wrong folder. Repeat Step 5 and confirm the project files are listed. |
| `.venv` creation or package install fails | Confirm Python is 3.10+, check internet access, remove only the incomplete `.venv` folder, then repeat Step 6. Do not remove the whole project. |
| `ModuleNotFoundError` | Run the install command from Step 6 again, then launch with the exact `.venv` Python command from Step 8. |
| `PINECONE_API_KEY not found` or `APIYI_API_KEY not found` | Confirm `.env` is in the project root and is not named `.env.txt`. Reopen it and fill both keys. |
| `invalid_api_key`, `401`, or authentication error | A key is wrong, expired, copied incompletely, or belongs to the wrong service. Create a new key, replace only its value in `.env`, save, and restart. |
| `429`, quota, balance, or billing error | Check APIYI credit and Pinecone plan usage in their consoles. Starting the chatbot, ingestion, and questions can all make provider calls. |
| Pinecone permission error | Create a Pinecone key for the selected project with permission to create an index and read/write/delete records. Replace the key in `.env`. |
| Pinecone region or plan error | Keep `PINECONE_CLOUD=aws` and `PINECONE_REGION=us-east-1` on Starter or Builder plans. |
| Pinecone dimension mismatch | An existing index with that name has a different dimension. Set `PINECONE_INDEX=my-first-rag-2`, save `.env`, restart, and re-ingest. |
| Pinecone index is not ready | Wait 30 seconds and launch again. First-time index creation is not always ready after the script's ten-second wait. |
| APIYI says model not found | Confirm the APIYI account can access `text-embedding-3-small` and `gpt-4o-mini`. Use the model ID shown in your APIYI account if access differs. |
| `ingest_files()` says folder not found | Create the exact `Files to insert (PDF or TXT)` folder in the project root using Step 9. |
| `ingest_files()` finds no files | Put PDF, TXT, or MD files directly inside the ingestion folder, not in a subfolder. |
| PDF stores zero chunks | The PDF may be an image scan, encrypted, corrupt, or have no extractable text. Use a searchable/OCR version. |
| Answer says information is missing after a successful ingest | Check `Total chunks stored`, wait 10–30 seconds for Pinecone consistency, then ask a more specific question. Confirm the startup index and namespace match the ingestion run. |
| Website ingestion fails or closes the program | Restart and try a public URL that loads without login. The script has a 30-second request timeout and does not catch every website network error. |
| Wrong or stale answer | Verify against the original file. Use a fresh namespace or carefully clear and re-ingest. This starter does not show citations or guarantee correctness. |
| School/company network blocks the connection | Try another network. Do not disable SSL verification. If the organization requires a proxy, ask its IT administrator for approved settings. |

If requesting help, share the command you ran and the error text, but redact both API keys. Never send the `.env` file.

## Configuration reference

Beginners should keep every value except the two keys unchanged.

| Variable | Required? | Purpose |
| --- | --- | --- |
| `APIYI_API_KEY` | Yes | Authenticates embedding and chat requests through APIYI. |
| `PINECONE_API_KEY` | Yes | Authenticates Pinecone index and vector operations. |
| `PINECONE_INDEX` | No | Index to reuse or create; default `my-first-rag`. |
| `PINECONE_CLOUD` | No | Cloud used only while creating a new index; default `aws`. |
| `PINECONE_REGION` | No | Region used only while creating a new index; default `us-east-1`. |
| `PINECONE_NAMESPACE` | No | Partition used for upsert, query, and `clearDB()`; default `__default__`. |
| `EMBED_MODEL` | No | APIYI embedding model; default `text-embedding-3-small`. |
| `EMBED_DIMENSIONS` | No | Vector size; default `1024`, which must match the index. |
| `CHAT_MODEL` | No | APIYI chat model; default `gpt-4o-mini`. |

The OpenAI Python package is used only as an OpenAI-compatible client. Requests go to `https://api.apiyi.com/v1`; this script does not read `OPENAI_API_KEY`.

## Important limitations

- Each chunk causes a separate embedding request and Pinecone write. Start with a small document to control cost.
- Each question causes an embedding request, a Pinecone query, and a chat request.
- Text is split by characters: 800 characters per chunk with 100 characters of overlap.
- The script retrieves at most three chunks and does not display sources, scores, or citations.
- It does not perform OCR, recursively scan folders, batch requests, deduplicate documents, retry failed requests, or provide multi-user security.
- Generated answers can be wrong. Check important answers against the original source.

## Verified references

This guide was checked against the repository code and these official sources in September 2026:

- [Python downloads](https://www.python.org/downloads/) and [`venv` documentation](https://docs.python.org/3/library/venv.html)
- [APIYI quick start](https://docs.apiyi.com/en/getting-started) and [embeddings API](https://docs.apiyi.com/api-reference/embeddings/create-embeddings)
- [Pinecone API-key instructions](https://docs.pinecone.io/guides/projects/manage-api-keys), [Python SDK](https://docs.pinecone.io/reference/sdks/python/overview), and [index limits](https://docs.pinecone.io/reference/api/database-limits)
- [pypdf text-extraction limitations](https://pypdf.readthedocs.io/en/latest/user/extract-text.html)
- [Jina Reader documentation](https://github.com/jina-ai/reader)
