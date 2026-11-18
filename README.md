# DotNetAI.LocalMaster

Local, offline AI demo for .NET 9. The API combines three things that all run on this machine:

- **Semantic Kernel** talks to a local [Ollama](https://ollama.com) chat model. A native C# plugin is registered on the kernel and can be invoked without calling the model.
- **Retrieval-augmented generation** chunks text, embeds it with a local Ollama embedding model, and searches an in-memory vector store.
- **ML.NET** trains a customer-feedback sentiment model from an in-memory sample catalog and serves predictions through `PredictionEnginePool`.

The orchestration endpoint classifies a message with ML.NET first, then asks the local chat model for a reply that matches that classification. No cloud API key is used. `appsettings.json` contains only the local Ollama address and model names.

## Projects

| Project | Role |
| --- | --- |
| `src/DotNetAI.Core` | Models, options, and service interfaces. |
| `src/DotNetAI.Infrastructure` | Semantic Kernel, in-memory vectors, and the ML.NET pipeline. |
| `src/DotNetAI.API` | HTTP endpoints and Swagger. |
| `DotNetAI.Docs` | Architecture and API notes. |

Dependencies are registered in one place: `AddLocalInfrastructureServices`.

## Prerequisites

- .NET 9 SDK
- Ollama, installed separately, listening on `http://localhost:11434`
- Two local models: `llama3` and `nomic-embed-text`

Install Ollama and pull both models on the machine that will run the API before calling chat, embedding, or orchestration. Sentiment classification and native plugin calls work with the API alone.

## Install Ollama and pull the models

### Windows

1. Download the Windows installer from [https://ollama.com/download](https://ollama.com/download) and run it. The installer starts the local server. You can also start it yourself:

   ```powershell
   ollama serve
   ```

2. Pull the chat model and the embedding model:

   ```powershell
   ollama pull llama3
   ollama pull nomic-embed-text
   ```

3. Confirm both models are present:

   ```powershell
   ollama list
   ```

### macOS and Linux

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama serve
ollama pull llama3
ollama pull nomic-embed-text
```

`llama3` is the chat model. `nomic-embed-text` produces the vectors used by retrieval. Both names are the defaults in `src/DotNetAI.API/appsettings.json`:

```json
"Ollama": {
  "Endpoint": "http://localhost:11434",
  "ChatModelId": "llama3",
  "EmbeddingModelId": "nomic-embed-text",
  "RequestTimeoutSeconds": 180
}
```

Change `ChatModelId` if you pulled a different tag, such as `llama3:latest` is matched automatically. A model id of `llama3` matches the tag `llama3:latest`.

## Run the API

From the repository root:

```powershell
dotnet build DotNetAI.LocalMaster.slnx
dotnet run --project src/DotNetAI.API --launch-profile http
```

Swagger is at [http://localhost:5233/swagger](http://localhost:5233/swagger). The HTTPS profile uses [https://localhost:7269/swagger](https://localhost:7269/swagger).

On startup the API trains the ML.NET sentiment model in memory and logs the training-sample count and training-set accuracy. That accuracy is measured on the same dummy samples used to fit the model. It shows that the pipeline trained. It is not a held-out evaluation.

## What to try

These calls do not need Ollama:

```powershell
Invoke-RestMethod http://localhost:5233/api/sentiment/model

Invoke-RestMethod http://localhost:5233/api/sentiment/classify `
  -Method Post -ContentType application/json `
  -Body '{"text":"This product is terrible and I want a refund."}'

Invoke-RestMethod http://localhost:5233/api/chat/plugins

Invoke-RestMethod http://localhost:5233/api/chat/plugins `
  -Method Post -ContentType "application/json" `
  -Body '{"functionName":"GetReturnPolicy"}'
```

After Ollama is running and both models are pulled:

```powershell
Invoke-RestMethod http://localhost:5233/api/health/ollama

Invoke-RestMethod http://localhost:5233/api/chat `
  -Method Post -ContentType application/json `
  -Body '{"message":"What is your return policy?","enablePlugins":true}'

Invoke-RestMethod http://localhost:5233/api/orchestration `
  -Method Post -ContentType application/json `
  -Body '{"message":"The jacket arrived damaged and I want a refund."}'
```

A sample policy document for retrieval is in `DotNetAI.Docs/sample-knowledge.md`. Post its text to `POST /api/rag/documents`, then query it with `POST /api/rag/query`.

## When Ollama is not running

- `GET /api/health/ollama` returns HTTP 200 with `isReachable: false` and a short explanation.
- Chat and embedding calls return HTTP 503 with a problem-details body. The message says the local server could not be reached.
- `POST /api/orchestration` still returns the ML.NET label. `languageModelUsed` is false and `reply` explains that the language model is unavailable.
- `POST /api/rag/query` against an empty store returns that explanation without calling Ollama.

## Configuration

| Setting | Purpose |
| --- | --- |
| `Ollama:Endpoint` | Local Ollama base address. |
| `Ollama:ChatModelId` | Model used by chat and orchestration. |
| `Ollama:EmbeddingModelId` | Model used to embed document chunks and questions. |
| `Ollama:RequestTimeoutSeconds` | Chat and embedding timeout, from 10 to 600 seconds. |
| `Rag:ChunkSize` | Target chunk length in characters. |
| `Rag:ChunkOverlap` | Characters repeated between chunks. |
| `Rag:TopK` | Default number of chunks returned by a search. |

There is no OpenAI, Azure OpenAI, or other cloud key in configuration.

## Further reading

- [Architecture](DotNetAI.Docs/Architecture.md) describes how a request moves through ML.NET, Semantic Kernel, and the API.
- [API](DotNetAI.Docs/API.md) lists every endpoint.
