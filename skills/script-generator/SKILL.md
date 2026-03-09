---
name: script-generator
description: |
  Generate a podcast or video script from markdown content using HeyGen's Node Gateway. Use when: (1) Converting a blog post or article into a video script, (2) Generating podcast scripts from written content, (3) Turning markdown into spoken-word scripts, (4) Working with HeyGen's /v1/node/execute endpoint for script generation.
allowed-tools: mcp__heygen__*
metadata:
  openclaw:
    requires:
      env:
        - HEYGEN_API_KEY
    primaryEnv: HEYGEN_API_KEY
---

# Script Generator (HeyGen Node Gateway)

Generate a podcast or video script from markdown content. Converts written content into a spoken-word script suitable for video or audio production.

## Authentication

All requests require the `X-Api-Key` header. Set the `HEYGEN_API_KEY` environment variable.

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "ScriptGeneratorNode", "input": {"markdown": "# My Blog Post\n\nThis is the content...", "length": 3}}'
```

## Default Workflow

1. Call `POST /v1/node/execute` with `node_type: "ScriptGeneratorNode"` and your markdown content
2. Receive a `workflow_id` in the response
3. Poll `GET /v1/node/status?workflow_id={id}` every 5 seconds until status is `completed`
4. Read the generated script from `output.script`

## Execute Script Generation

### Endpoint

`POST https://api.heygen.com/v1/node/execute`

### Request Fields

| Field | Type | Req | Description |
|-------|------|:---:|-------------|
| `node_type` | string | Y | Must be `"ScriptGeneratorNode"` |
| `input.markdown` | string | Y | Markdown content to convert into a script |
| `input.script_type` | string | | Script style — `"podcast"` (default) |
| `input.length` | integer | Y | Target length of the script in minutes |
| `input.language` | string | | Language code (default: `"en"`) |

### curl

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "ScriptGeneratorNode",
    "input": {
      "markdown": "# AI Video Generation\n\nAI-powered video generation is transforming content creation. Companies can now produce professional videos in minutes instead of days...",
      "script_type": "podcast",
      "length": 5,
      "language": "en"
    }
  }'
```

### TypeScript

```typescript
interface ScriptGeneratorInput {
  markdown: string;
  script_type?: "podcast";
  length: number;
  language?: string;
}

async function generateScript(input: ScriptGeneratorInput): Promise<string> {
  const response = await fetch("https://api.heygen.com/v1/node/execute", {
    method: "POST",
    headers: {
      "X-Api-Key": process.env.HEYGEN_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      node_type: "ScriptGeneratorNode",
      input,
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

def generate_script(
    markdown: str,
    length: int,
    script_type: str = "podcast",
    language: str = "en",
) -> str:
    response = requests.post(
        "https://api.heygen.com/v1/node/execute",
        headers={
            "X-Api-Key": os.environ["HEYGEN_API_KEY"],
            "Content-Type": "application/json",
        },
        json={
            "node_type": "ScriptGeneratorNode",
            "input": {
                "markdown": markdown,
                "script_type": script_type,
                "length": length,
                "language": language,
            },
        },
    )

    return response.json()["data"]["workflow_id"]
```

### Response Format

```json
{
  "data": {
    "workflow_id": "node-gw-s1c2r3i4",
    "status": "submitted"
  }
}
```

## Check Status

### Endpoint

`GET https://api.heygen.com/v1/node/status?workflow_id={workflow_id}`

### curl

```bash
curl -X GET "https://api.heygen.com/v1/node/status?workflow_id=node-gw-s1c2r3i4" \
  -H "X-Api-Key: $HEYGEN_API_KEY"
```

### Response Format (Completed)

```json
{
  "data": {
    "workflow_id": "node-gw-s1c2r3i4",
    "status": "completed",
    "output": {
      "script": "Welcome to today's episode. We're diving into the world of AI video generation — a technology that's completely changing how companies create content..."
    }
  }
}
```

## Polling for Completion

```typescript
async function generateScriptAndWait(
  input: ScriptGeneratorInput,
  maxWaitMs = 120000,
  pollIntervalMs = 5000
): Promise<string> {
  const workflowId = await generateScript(input);
  console.log(`Submitted script generation: ${workflowId}`);

  const startTime = Date.now();
  while (Date.now() - startTime < maxWaitMs) {
    const response = await fetch(
      `https://api.heygen.com/v1/node/status?workflow_id=${workflowId}`,
      { headers: { "X-Api-Key": process.env.HEYGEN_API_KEY! } }
    );
    const { data } = await response.json();

    switch (data.status) {
      case "completed":
        return data.output.script;
      case "failed":
        throw new Error(data.error?.message || "Script generation failed");
      case "not_found":
        throw new Error("Workflow not found");
      default:
        await new Promise((r) => setTimeout(r, pollIntervalMs));
    }
  }

  throw new Error("Script generation timed out");
}
```

## Usage Examples

### Blog Post to Podcast Script

```json
{
  "node_type": "ScriptGeneratorNode",
  "input": {
    "markdown": "# 5 Ways AI is Changing Marketing\n\n## 1. Personalized Content at Scale\nAI enables...\n\n## 2. Automated Video Production\n...",
    "script_type": "podcast",
    "length": 10,
    "language": "en"
  }
}
```

### Short Product Script

```json
{
  "node_type": "ScriptGeneratorNode",
  "input": {
    "markdown": "# Product Launch: HeyGen API v2\n\nNew features include node gateway, batch processing, and webhook support.",
    "length": 2
  }
}
```

## Best Practices

1. **Use structured markdown** — headings, lists, and sections produce better scripts
2. **Set realistic lengths** — a 5-minute script needs substantial source material
3. **Chain with TTS or avatar** — feed the generated script into text-to-speech or avatar video
4. **Language matters** — set `language` to match your source content for best results
5. **Script generation takes 15-30 seconds** depending on content length
