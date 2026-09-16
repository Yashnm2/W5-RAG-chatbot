# How `Starter.py` works

`Starter.py` is a command-line Retrieval-Augmented Generation (RAG) chatbot. It turns local documents or web pages into searchable text, stores that text in Pinecone, and uses relevant passages to answer questions through APIYI's OpenAI-compatible API.

## The overall flow

```text
PDF, TXT, Markdown, or web page
            ↓
       Extract text
            ↓
  Split into overlapping chunks
            ↓
 Create an embedding for each chunk
            ↓
 Store vectors and original text in Pinecone

User question
      ↓
Create a question embedding
      ↓
Retrieve the three closest chunks from Pinecone
      ↓
Send question + retrieved context to the chat model
      ↓
Print the answer in the terminal
```

This is called RAG because the program first **retrieves** useful information and then asks a model to **generate** an answer from it.

## Startup and configuration

When the script starts, it:

1. Loads values from `.env` with `python-dotenv`.
2. Checks that `PINECONE_API_KEY` and `APIYI_API_KEY` exist.
3. Creates an OpenAI-compatible client whose requests go to `https://api.apiyi.com/v1`.
4. Connects to Pinecone.
5. Creates the configured Pinecone index if it does not already exist, using cosine similarity and the configured embedding dimension.
6. Starts the interactive terminal chatbot through `run_chatbot()`.

The main settings are:

| Environment variable | Default | Purpose |
| --- | --- | --- |
| `PINECONE_API_KEY` | Required | Authenticates with Pinecone. |
| `APIYI_API_KEY` | Required | Authenticates embedding and chat requests through APIYI. |
| `PINECONE_INDEX` | `my-first-rag` | Pinecone index to use or create. |
| `PINECONE_CLOUD` | `aws` | Cloud used when creating a new index. |
| `PINECONE_REGION` | `us-east-1` | Region used when creating a new index. |
| `PINECONE_NAMESPACE` | `__default__` | Namespace used for all reads, writes, and deletes. |
| `EMBED_MODEL` | `text-embedding-3-small` | Model used to turn text into vectors. |
| `EMBED_DIMENSIONS` | `1024` | Required vector length for both the embedding model and Pinecone index. |
| `CHAT_MODEL` | `gpt-4.1-nano` | Model used to write the final answer. |

The embedding dimension must match the Pinecone index dimension. The script also checks every returned embedding and raises an error if its length is wrong.

## Function reference

### `create_embedding(text)`

Sends text to the configured embedding model through APIYI and returns its numeric vector. Both stored passages and user questions use this function, so Pinecone can compare them in the same vector space.

### `store_in_pinecone(text, source_name, chunk_id)`

Embeds one passage and upserts it into Pinecone. Its vector ID is formed as `source_name_chunk_id`, and its metadata contains the original text and source name.

Because the ID is deterministic, ingesting the same source with the same chunk numbers overwrites matching vector IDs. If a newly ingested version contains fewer chunks, older higher-numbered chunks are not automatically removed.

### `read_file(file_path)`

Reads a local file:

- PDF files are processed page by page with `pypdf.PdfReader`.
- Other files are read as UTF-8 text. In normal use, the folder scanner selects `.txt`, `.md`, and `.pdf` files.

It returns an empty string when the file is missing or cannot be read.

### `chunk_text(text, chunk_size=800, overlap=100)`

Splits text by character count into chunks of up to 800 characters. Consecutive chunks share 100 characters, which helps preserve context across boundaries. It rejects invalid values such as a non-positive chunk size or an overlap equal to or larger than the chunk size.

Chunking is character-based, so it can split in the middle of a sentence or word.

### `store_file_in_pinecone(file_path, chunk_size=800, overlap=100)`

Combines the file-processing steps: read the file, derive a source name from its filename, chunk the text, and store every chunk in Pinecone. It returns the number of stored chunks.

### `ingest_files_from_folder(folder_path="Files to insert (PDF or TXT)")`

Scans the named folder for PDF, TXT, and Markdown files and sends each file through `store_file_in_pinecone`. It prints an ingestion summary and reminds the user to move processed files so they are not ingested again. Its return value is the number of files processed, not the number of chunks.

### `scrape_website(url)`

Prefixes the supplied URL with `https://r.jina.ai/` and downloads Jina Reader's text/Markdown representation of the page. The request has a 30-second timeout. A successful HTTP response returns its text; a non-200 response returns an empty string.

### `retrieve_from_pinecone(query, top_k=3)`

Embeds the question and performs a cosine-similarity query in the configured Pinecone namespace. It requests metadata and returns the original text from the best matching chunks. The default is three results.

### `get_relevant_context(question, top_k=3)`

Calls the retrieval function and joins the returned chunks with blank lines to form one context string.

### `chat_with_rag(user_question)`

Retrieves three relevant chunks, inserts them and the user's question into a prompt, and calls the configured chat model through APIYI. The prompt tells the model to say it lacks the information when the answer is not present in the retrieved context. Responses use a temperature of `0.7` and a maximum of 500 tokens.

### `clear_database()`

Deletes every vector in the configured Pinecone namespace. The CLI asks for an exact `yes` confirmation before calling it. The deletion cannot be undone.

### `run_chatbot()`

Runs the terminal input loop and recognizes these inputs:

| Input | Result |
| --- | --- |
| Any normal text | Runs the RAG question-answering flow. |
| `ingest_files()` | Ingests supported files from `Files to insert (PDF or TXT)`. |
| `scrape_website()` | Prompts for a URL, scrapes it, chunks it, and stores it. |
| `clearDB()` | Requests confirmation, then clears the namespace. |
| `quit`, `exit`, or `bye` | Ends the program. |

The special commands are case-sensitive except for the exit words. Errors while answering a question are caught so the chatbot can continue running.

## Running the script

Install the dependencies, create and fill in `.env`, then run:

```bash
python Starter.py
```

## Important limitations

- Files, web content, and questions leave the local computer: APIYI receives text for embeddings and chat, Pinecone stores chunks and vectors, and Jina Reader receives URLs used for web ingestion.
- The chatbot returns an answer without source citations or similarity scores.
- Retrieved passages are selected by semantic similarity, but the script does not enforce a minimum relevance score.
- The prompt encourages the model to rely on context, but it cannot guarantee a correct or non-hallucinated answer.
- Vector IDs can collide when different files have the same base filename or different web URLs produce the same normalized domain.
- Website fetching handles non-200 responses but does not catch request exceptions inside `scrape_website()` itself.
- The program is a learning-oriented, single-user terminal application; it does not provide authentication, access control, or production deployment features.
