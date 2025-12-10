# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

geminicli2api is a FastAPI proxy server that converts Google's Gemini CLI tool into both OpenAI-compatible and native Gemini API endpoints. It allows using Google's free Gemini API quota through familiar OpenAI API interfaces.

## Running the Server

```bash
# Install dependencies
pip install -r requirements.txt

# Run locally (port 8888)
python run.py

# Run for Hugging Face Spaces (port 7860)
python app.py

# Docker
docker build -t geminicli2api .
docker run -p 8888:8888 -e GEMINI_AUTH_PASSWORD=your_password -e GEMINI_CREDENTIALS='{}' geminicli2api
```

## Environment Variables

- `GEMINI_AUTH_PASSWORD` (required): Authentication password for API access
- `GEMINI_CREDENTIALS`: JSON string with Google OAuth credentials (client_id, client_secret, token, refresh_token)
- `GOOGLE_APPLICATION_CREDENTIALS`: Path to OAuth credentials file
- `GOOGLE_CLOUD_PROJECT` / `GEMINI_PROJECT_ID`: Google Cloud project ID
- `PORT`: Server port (default: 8888 for run.py, 7860 for app.py)

## Architecture

### Request Flow

1. **Incoming Request** → Routes (`openai_routes.py` or `gemini_routes.py`)
2. **Authentication** → `auth.py:authenticate_user()` validates credentials
3. **Transformation** (OpenAI only) → `openai_transformers.py` converts OpenAI format to Gemini format
4. **API Call** → `google_api_client.py:send_gemini_request()` sends to Google's internal endpoint
5. **Response** → Transformed back (OpenAI) or passed through (native Gemini)

### Key Modules

- **`src/main.py`**: FastAPI app setup, CORS, startup events, health check
- **`src/openai_routes.py`**: `/v1/chat/completions` and `/v1/models` endpoints
- **`src/gemini_routes.py`**: `/v1beta/models/*` native Gemini proxy endpoints
- **`src/openai_transformers.py`**: Bidirectional conversion between OpenAI and Gemini formats
- **`src/google_api_client.py`**: Handles actual HTTP calls to `cloudcode-pa.googleapis.com`
- **`src/auth.py`**: OAuth2 flow, credential management, request authentication
- **`src/config.py`**: Model definitions, safety settings, thinking budget configuration
- **`src/models.py`**: Pydantic models for request/response validation

### Model Variants System

Models in `config.py` have automatic variant generation:
- Base models: `gemini-2.5-pro`, `gemini-2.5-flash`, etc.
- `-search` suffix: Enables Google Search grounding
- `-nothinking` suffix: Minimizes reasoning tokens
- `-maxthinking` suffix: Maximizes thinking budget

Helper functions in `config.py`:
- `get_base_model_name()`: Strips variant suffixes
- `is_search_model()`, `is_nothinking_model()`, `is_maxthinking_model()`: Check variant type
- `get_thinking_budget()`: Returns appropriate token budget per variant

### Thinking/Reasoning Support

The proxy exposes Gemini's thinking tokens through OpenAI-compatible `reasoning_content` field:
- Request: `reasoning_effort` parameter (minimal/low/medium/high) maps to thinking budgets
- Response: `reasoning_content` in message/delta contains thinking tokens

### Image Handling

`openai_transformers.py` handles multimodal:
- Converts OpenAI `image_url` parts to Gemini `inlineData`
- Extracts data URIs from Markdown images `![](data:image/...;base64,...)`
- Response images converted back to Markdown data URIs

## API Endpoints

### OpenAI-Compatible
- `POST /v1/chat/completions` - Chat completions (streaming & non-streaming)
- `GET /v1/models` - List available models

### Native Gemini
- `GET /v1beta/models` - List Gemini models
- `POST /v1beta/models/{model}:generateContent` - Generate content
- `POST /v1beta/models/{model}:streamGenerateContent` - Stream content

### Authentication Methods
All require `GEMINI_AUTH_PASSWORD`:
- Bearer token: `Authorization: Bearer {password}`
- Basic auth: `Authorization: Basic base64(user:password)`
- Query param: `?key={password}`
- Google header: `x-goog-api-key: {password}`
