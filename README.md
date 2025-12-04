# iBOS AI Blog Generator (n8n Workflow)

This repository contains an **n8n workflow** that generates a fully structured **SEO-friendly blog article + cover image + tags** for iBOS (International British Online School) via a single HTTP request.

The workflow exposes a **`/blog` Webhook**, calls several OpenAI models, and returns a clean JSON object ready to be stored in your CMS or backend.

---

## Features

- **HTTP Webhook input** (`POST /webhook/blog`)
- Generates a **full blog article** based on:
  - `title`
  - `keywords[]`
  - `tone`
  - `aspect`
- Produces **clean, SEO-friendly HTML content**:
  - Only content tags like `<article>`, `<section>`, `<h1–h3>`, `<p>`, `<ul>`, `<ol>`, etc.
  - No `<html>`, `<head>`, `<body>`, `<script>`, etc. (ready to embed directly)
- Extracts **SEO tags** from the generated article (no extra commentary)
- Generates a **16:9 cover image URL** via OpenAI image API
- Returns a **single JSON response**:

  ```json
  {
    "title": "...",
    "content": "<article>...</article>",
    "cover": "https://...",
    "tag": "...",
    "created_at": "2025-12-03T...",
    "initiated_at": "2025-12-03T...",
    "id": "blog-2025-0001"
  }
  ```

---

## Workflow Overview

The imported workflow is named something like **`My workflow`** in n8n.

Main nodes:

1. **Webhook**
   - Method: `POST`
   - Path: `blog`
   - Mode: `responseMode = responseNode` (final response comes from a later node)

2. **Edit Fields (Set)**
   - Extracts and normalizes input fields:
     - `title       = {{$json.body.title}}`
     - `keywords    = {{$json.body.keywords}}` (array)
     - `tone        = {{$json.body.tone}}`
     - `aspect      = {{$json.body.aspect}}`
     - `id          = {{$json.body.id}}`
     - `initiatd_at = {{$now}}` (typo kept as in workflow)

3. **OpenAI Chat Model**
   - Provides the base language model (for example `gpt-5.1`) for the next node.

4. **Basic LLM Chain**
   - System prompt describes how to:
     - Write an iBOS blog article using `title`, `aspect`, `tone`, and `keywords`.
     - Produce **only article HTML content**, without global HTML boilerplate.

5. **Message a model** (Tags Generator)
   - Model: `gpt-4o-mini`
   - Role: SEO specialist.
   - Instruction: “get me best tags from this blog without any extra explanation and get me just tags”.

6. **Message a model1** (Image Prompt Generator)
   - Model: `gpt-4.1`
   - Role: Image prompt engineer.
   - Instruction: “make prompt for image generation for following blog without any extra explanation”.

7. **Generate an image**
   - Uses OpenAI image API.
   - Input: prompt from **Message a model1** + blog content.
   - Output: image URL (16:9 style, configured in node).

8. **Merge**
   - Merges the tag output and image URL into one flow.

9. **Edit Fields1 (Set final JSON)**
   - Builds the final JSON object:

     ```js
     {
       title:        $node["Webhook"].json.body.title,
       content:      $node["Basic LLM Chain"].json.text,
       cover:        $item(1).$node["Merge"].json.url,
       tag:          $node["Merge"].json.output[0].content[0].text,
       created_at:   $now,
       initiated_at: $node["Edit Fields"].json.initiatd_at,
       id:           $node["Edit Fields"].json.id
     }
     ```

10. **Respond to Webhook**
    - Responds with JSON.
    - HTTP status code: `200`.

---

## Requirements

- **n8n** instance (self-hosted or cloud)
- Valid **OpenAI API key**
- n8n OpenAI credentials configured for:
  - `OpenAI Chat Model`
  - `Message a model`
  - `Message a model1`
  - `Generate an image`

---

## Installation & Setup

1. **Import the Workflow**

   - In n8n, go to **Workflows → Import from File**.
   - Select `My workflow (1).json` (or the exported workflow file) and import.

2. **Configure OpenAI Credentials**

   - Go to **Credentials → New → OpenAI API**.
   - Create a credential (for example named `OpenAi account`).
   - Make sure each OpenAI-based node in the workflow points to this credential.

3. **Activate the Workflow**

   - Turn the workflow **Active**.
   - This activates the production webhook endpoint.

---

## Webhook Endpoints

### Test Mode (inside n8n)

When you click **“Execute Workflow”** in the editor, n8n provides a temporary URL like:

```text
POST http://localhost:5678/webhook-test/blog
```

- Valid for **one request** after clicking “Execute”.
- Useful for quick debugging.

### Production Mode

With the workflow **Active**, the production endpoint is:

```text
POST http://<YOUR_N8N_HOST>:5678/webhook/blog
```

Examples:

- Local:
  ```text
  POST http://localhost:5678/webhook/blog
  ```
- Remote (via reverse proxy / domain):
  ```text
  POST https://n8n.your-domain.com/webhook/blog
  ```

(Adjust host and port to your deployment.)

---

## Request Format

**HTTP**

```http
POST /webhook/blog
Content-Type: application/json
```

**Body**

```json
{
  "title": "British Online School – Quality Education from Anywhere",
  "keywords": [
    "british school",
    "online learning",
    "IGCSE",
    "A-Level",
    "IBOS"
  ],
  "tone": "friendly",
  "aspect": "learning",
  "id": "blog-2025-0001"
}
```

### Parameters

- `title` (string) – Blog post title.
- `keywords` (array of strings) – SEO keywords to be **used in the article**.
- `tone` (string) – Writing tone, e.g. `friendly`, `professional`, `parent-focused`, etc.
- `aspect` (string) – Main angle of the article, e.g. `learning`, `admissions`, `fees`, `curriculum`.
- `id` (string) – Custom identifier you assign (for DB mapping, tracking, etc).

---

## Response Format

Example response:

```json
{
  "title": "British Online School – Quality Education from Anywhere",
  "content": "<article><h1>...</h1><p>...</p>...</article>",
  "cover": "https://api.openai.com/...",
  "tag": "- british online school\n- online learning\n- IGCSE\n- A-Level\n- international students",
  "created_at": "2025-12-03T14:35:12.345Z",
  "initiated_at": "2025-12-03T14:35:10.123Z",
  "id": "blog-2025-0001"
}
```

- `content` – Full SEO-optimized HTML article (no `<html>`, `<head>`, `<body>`).
- `cover` – URL for the generated 16:9 cover image.
- `tag` – List of tags/keywords as plain text (format depends on model; typically bullet list or comma-separated).
- `created_at` – Time when the final JSON was built.
- `initiated_at` – Time when processing started (from `Edit Fields` node).
- `id` – Echo of the input `id`.

---

## Quick Test with cURL

```bash
curl -X POST "http://localhost:5678/webhook/blog" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "british school quality",
    "keywords": ["english", "british", "school"],
    "tone": "friendly",
    "aspect": "learning",
    "id": "blog-test-001"
  }'
```

You should receive a JSON response containing the generated blog, tags, and cover image URL.

---

## Extending the Workflow

Some suggestions for future improvements:

- **Multi-language support**  
  Add a `language` field and branch prompts for different locales.

- **Automatic persistence**  
  Save results directly to:
  - A database (PostgreSQL / MySQL / MongoDB)
  - Google Sheets / Airtable / Notion

- **Auto-publishing**  
  Add nodes to publish directly to:
  - WordPress
  - Webflow
  - Custom CMS via REST API

- **Configurable summary depth**  
  Let the client send a `summary_level` (e.g. `long`, `medium`, `short`) and adjust the LLM prompt accordingly.

- **Analytics and logging**  
  Log:
  - Generation time
  - Prompt tokens
  - Response tokens
  - Any errors for monitoring & debugging.

---

## License

This workflow is currently intended for internal use (e.g., iBOS or your own organization).  
Update this section with your preferred license (MIT, proprietary, etc.) before publishing the repository.
