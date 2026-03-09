---
name: pdf-extraction
description: |
  Extract text content from a PDF document using HeyGen's Node Gateway. Use when: (1) Extracting text from PDF files, (2) Converting a PDF URL to plain text, (3) Getting PDF content for summarization or script creation, (4) Working with HeyGen's /v1/node/execute endpoint for PDF extraction.
allowed-tools: mcp__heygen__*
metadata:
  openclaw:
    requires:
      env:
        - HEYGEN_API_KEY
    primaryEnv: HEYGEN_API_KEY
---

# PDF Extraction (HeyGen Node Gateway)

Extract text content from a PDF document URL. Useful for turning documents into content that can be summarized, scripted, or used as input for video generation.

## Authentication

All requests require the `X-Api-Key` header. Set the `HEYGEN_API_KEY` environment variable.

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "PDFExtractionNode", "input": {"url": "https://example.com/report.pdf"}}'
```

## Default Workflow

1. Call `POST /v1/node/execute` with `node_type: "PDFExtractionNode"` and the PDF URL
2. Receive a `workflow_id` in the response
3. Poll `GET /v1/node/status?workflow_id={id}` every 3 seconds until status is `completed`
4. Read the extracted text from `output.text`

## Execute Extraction

### Endpoint

`POST https://api.heygen.com/v1/node/execute`

### Request Fields

| Field | Type | Req | Description |
|-------|------|:---:|-------------|
| `node_type` | string | Y | Must be `"PDFExtractionNode"` |
| `input.url` | string | Y | URL of the PDF document to extract text from |

### curl

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "PDFExtractionNode",
    "input": {
      "url": "https://example.com/quarterly-report.pdf"
    }
  }'
```

### TypeScript

```typescript
async function extractPDF(url: string): Promise<string> {
  const response = await fetch("https://api.heygen.com/v1/node/execute", {
    method: "POST",
    headers: {
      "X-Api-Key": process.env.HEYGEN_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      node_type: "PDFExtractionNode",
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

def extract_pdf(url: str) -> str:
    response = requests.post(
        "https://api.heygen.com/v1/node/execute",
        headers={
            "X-Api-Key": os.environ["HEYGEN_API_KEY"],
            "Content-Type": "application/json",
        },
        json={"node_type": "PDFExtractionNode", "input": {"url": url}},
    )

    return response.json()["data"]["workflow_id"]
```

### Response Format (Completed)

```json
{
  "data": {
    "workflow_id": "node-gw-p1d2f3e4",
    "status": "completed",
    "output": {
      "text": "Quarterly Report Q4 2025\n\nRevenue increased by 45%..."
    }
  }
}
```

## Best Practices

1. **Use publicly accessible PDF URLs** — the URL must be directly downloadable
2. **Extraction is fast** — typically completes within 5-15 seconds depending on PDF size
3. **Chain with LLM + Script Generator** — extract PDF, summarize with LLM, generate a video script
4. **Large PDFs produce long text** — consider using LLMNode to summarize before further processing
