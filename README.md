# AI Knowledge Assistant

An AI-powered document question-answering application built with Flask, LangChain, OpenAI, FAISS, and a browser-based frontend. Upload PDF or TXT documents, build a searchable knowledge base, and ask questions about the uploaded content.

## Features

- Upload one or multiple PDF and TXT files.
- Drag and drop files or select them from the file picker.
- Secure uploaded filenames with Werkzeug.
- Extract text from PDFs and text files.
- Split documents into overlapping chunks for retrieval.
- Generate local embeddings with `sentence-transformers/all-MiniLM-L6-v2`.
- Store embeddings in a FAISS vector index in memory.
- Answer questions with OpenAI's `gpt-4o-mini` model.
- Display uploaded documents and file sizes.
- Delete individual documents or clear all documents.
- Return source document paths with answers.
- CORS support for the separate frontend development server.
- Health-check endpoint for local verification.

## Project Structure

```text
Project/
├── backend/
│   ├── app.py
│   ├── requirements.txt
│   └── .env                  # Create or update locally; do not commit
├── frontend/
│   └── index.html
├── uploads/                  # Created automatically by the backend
└── README.md
```

## Requirements

- Python 3.10 or newer
- An OpenAI API key with API billing/credits enabled
- Internet access on the first run to download Python packages and the embedding model
- A modern browser such as Chrome, Edge, Firefox, or Safari

## Configuration

Create or edit `backend/.env` and add your OpenAI API key:

```env
OPENAI_API_KEY=sk-your-real-openai-key
```

Do not paste the key directly into `backend/app.py`, `frontend/index.html`, or any source-controlled file. Do not share the real key or commit `.env`.

The backend reads the key with:

```python
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
```

The key is required when documents are processed because that is when the OpenAI chat model is initialized. The backend can still start and report healthy before a key is configured.

## Installation

From the project root, install the backend dependencies:

```bash
cd backend
python3 -m pip install -r requirements.txt
```

If you use a virtual environment, create and activate it before installing:

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
```

On Windows PowerShell, activate the environment with:

```powershell
.venv\Scripts\Activate.ps1
```

## Running the Application

The application uses two local development servers.

### 1. Start the backend

Open a terminal in the project root:

```bash
cd backend
python3 app.py
```

The Flask API runs at:

```text
http://localhost:5001
```

### 2. Start the frontend

Open a second terminal in the project root:

```bash
cd frontend
python3 -m http.server 8000
```

Open the application in a browser:

```text
http://localhost:8000
```

The frontend is configured to call the backend at `http://localhost:5001`.

## Using the Application

1. Open `http://localhost:8000`.
2. Select one or more `.pdf` or `.txt` files, or drag them into the upload area.
3. Wait for the documents to be processed.
4. Enter a question about the uploaded content.
5. Click **Send** or press Enter.
6. Review the answer and source documents.

Multiple files can be selected on macOS with Command-click or Shift-click. On Windows and Linux, use Ctrl-click or Shift-click.

The backend accepts a maximum request size of 16 MB. This limit applies to the upload request as a whole, not separately to each file.

## API Reference

All endpoints are served from `http://localhost:5001`.

### `GET /health`

Returns backend status and whether a QA chain has been created.

Example:

```bash
curl http://localhost:5001/health
```

Response:

```json
{
  "status": "healthy",
  "documents_loaded": false
}
```

### `POST /upload`

Uploads and processes one or more documents.

For multiple files, send repeated `files` fields:

```bash
curl -X POST http://localhost:5001/upload \
  -F "files=@document-one.pdf" \
  -F "files=@document-two.txt"
```

The legacy single-file field `file` is also accepted:

```bash
curl -X POST http://localhost:5001/upload \
  -F "file=@document.pdf"
```

Supported extensions are `.pdf` and `.txt`. The response includes successfully uploaded filenames and any per-file validation warnings.

### `POST /query`

Asks a question about the processed documents.

```bash
curl -X POST http://localhost:5001/query \
  -H "Content-Type: application/json" \
  -d '{"question":"What is this document about?"}'
```

Example response:

```json
{
  "answer": "The document is about ...",
  "sources": [
    "/path/to/uploads/document.pdf"
  ]
}
```

This endpoint returns an error until at least one document has been successfully processed.

### `GET /documents`

Lists uploaded documents and their sizes:

```bash
curl http://localhost:5001/documents
```

### `DELETE /delete/<filename>`

Deletes one uploaded document and rebuilds the knowledge base:

```bash
curl -X DELETE http://localhost:5001/delete/document.pdf
```

### `POST /clear`

Deletes all uploaded documents and clears the in-memory vector store and QA chain:

```bash
curl -X POST http://localhost:5001/clear
```

## How It Works

1. The frontend sends selected files to `POST /upload`.
2. Flask saves valid files to the root-level `uploads/` directory.
3. `PyPDFLoader` or `TextLoader` extracts the document text.
4. `RecursiveCharacterTextSplitter` divides text into chunks of 1,000 characters with 200-character overlap.
5. Hugging Face's `all-MiniLM-L6-v2` model creates embeddings locally.
6. FAISS indexes the embeddings in memory.
7. LangChain creates a retrieval QA chain using OpenAI `gpt-4o-mini`.
8. A query retrieves the three most relevant chunks and sends them to the model as context.
9. The response includes the generated answer and source metadata.

The FAISS index and QA chain are held in memory. Restarting the backend clears them, although uploaded files remain in `uploads/` and are not automatically reprocessed on startup.

## Troubleshooting

### `Invalid API Key`

Check that `backend/.env` contains a real, active key:

```env
OPENAI_API_KEY=sk-your-real-openai-key
```

Then restart the backend. Do not include placeholder text such as `sk-your-openai-api-key`.

### `OPENAI_API_KEY not found`

Confirm that:

- The file is named `.env`.
- It is located inside `backend/`.
- The variable is spelled exactly `OPENAI_API_KEY`.
- The backend was restarted after editing the file.

### `No documents uploaded yet`

Upload and successfully process at least one supported PDF or TXT file before querying.

### The frontend cannot connect to the backend

Confirm that both servers are running:

```bash
curl http://localhost:5001/health
curl -I http://localhost:8000
```

If the backend is running on another port, update the `API_URL` constant in `frontend/index.html`.

### First upload is slow

The first upload may take longer because Python packages and the sentence-transformer embedding model may need to be downloaded and loaded into memory.

### Upload processing fails

Check the backend terminal for the detailed exception. Confirm that:

- The file is a valid PDF or UTF-8 text file.
- The total request is below 16 MB.
- The OpenAI key is configured and has available usage.
- The machine has enough memory for the embedding model and FAISS index.

## Security and Production Notes

This project is configured for local development and is not production-hardened.

- Never commit `backend/.env` or expose the OpenAI API key in frontend code.
- Restrict CORS origins before deploying publicly.
- Replace Flask's development server with a production WSGI server.
- Add authentication and authorization before exposing upload, delete, clear, or query endpoints.
- Add persistent vector-store storage if indexes must survive restarts.
- Add stronger file validation, malware scanning, rate limiting, and per-user storage isolation for untrusted users.
- Avoid returning local filesystem paths in public API responses.
- Review and rotate the API key if it is ever exposed.

## Development Checks

Validate backend syntax without creating a Python bytecode cache:

```bash
cd backend
python3 -c "import ast, pathlib; ast.parse(pathlib.Path('app.py').read_text()); print('backend syntax valid')"
```

Verify the running backend:

```bash
curl http://localhost:5001/health
```

