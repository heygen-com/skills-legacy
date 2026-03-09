---
name: website-extraction
description: |
  Extract text content from a website URL using HeyGen's Node Gateway. Use when: (1) Scraping or extracting text from a web page, (2) Converting a URL to plain text content, (3) Getting website content for summarization or video script creation, (4) Working with HeyGen's /v1/node/execute endpoint for web extraction.
allowed-tools: mcp__heygen__*
metadata:
  openclaw:
    requires:
      env:
        - HEYGEN_API_KEY
    primaryEnv: HEYGEN_API_KEY
---

# Website Extraction (HeyGen Node Gateway)

Extract text content from a website URL. Useful for turning web pages into content that can be summarized, scripted, or used as input for video generation.

## Authentication

All requests require the `X-Api-Key` header. Set the `HEYGEN_API_KEY` environment variable.

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "WebsiteExtractionNode", "input": {"url": "https://example.com/blog-post"}}'
```

## Default Workflow

1. Call `POST /v1/node/execute` with `node_type: "WebsiteExtractionNode"` and the URL
2. Receive a `workflow_id` in the response
3. Poll `GET /v1/node/status?workflow_id={id}` every 3 seconds until status is `completed`
4. Read the extracted text from `output.text`

## Execute Extraction

### Endpoint

`POST https://api.heygen.com/v1/node/execute`

### Request Fields

| Field | Type | Req | Description |
|-------|------|:---:|-------------|
| `node_type` | string | Y | Must be `"WebsiteExtractionNode"` |
| `input.url` | string | Y | URL of the web page to extract text from |

### curl

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "WebsiteExtractionNode",
    "input": {
      "url": "https://example.com/product-page"
    }
  }'
```

### TypeScript

```typescript
async function extractWebsite(url: string): Promise<string> {
  const response = await fetch("https://api.heygen.com/v1/node/execute", {
    method: "POST",
    headers: {
      "X-Api-Key": process.env.HEYGEN_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      node_type: "WebsiteExtractionNode",
      input: { url },
    }),
  });

  const json = await response.json();
  return json.data.workflow_id;
}
```

### Python

```python
import requests
import os

def extract_website(url: str) -> str:
    response = requests.post(
        "https://api.heygen.com/v1/node/execute",
        headers={
            "X-Api-Key": os.environ["HEYGEN_API_KEY"],
            "Content-Type": "application/json",
        },
        json={"node_type": "WebsiteExtractionNode", "input": {"url": url}},
    )

    return response.json()["data"]["workflow_id"]
```

### Response Format (Completed)

```json
{
  "data": {
    "workflow_id": "node-gw-w3b5i6t7",
    "status": "completed",
    "output": {
      "text": "Welcome to our product page. Our AI-powered platform enables..."
    }
  }
}
```

## Best Practices

1. **Use publicly accessible URLs** — pages behind auth walls or paywalls may not extract correctly
2. **Extraction is fast** — typically completes within 5-10 seconds; poll every 3 seconds
3. **Chain with LLM** — extract a web page, then use LLMNode to summarize or create a video script
4. **Check output length** — very complex pages may produce long text that needs trimming
