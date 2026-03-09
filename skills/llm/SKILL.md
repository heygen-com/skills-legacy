---
name: llm
description: |
  Generate text using a large language model via HeyGen's Node Gateway. Use when: (1) Generating text from a prompt, (2) Using AI to write copy, scripts, or content, (3) Running LLM inference through HeyGen's API, (4) Working with HeyGen's /v1/node/execute endpoint for text generation.
allowed-tools: mcp__heygen__*
metadata:
  openclaw:
    requires:
      env:
        - HEYGEN_API_KEY
    primaryEnv: HEYGEN_API_KEY
---

# LLM Text Generation (HeyGen Node Gateway)

Generate text using a large language model with a prompt and optional template variables.

## Authentication

All requests require the `X-Api-Key` header. Set the `HEYGEN_API_KEY` environment variable.

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "LLMNode", "input": {"prompt": "Write a product tagline for a video editing tool"}}'
```

## Default Workflow

1. Call `POST /v1/node/execute` with `node_type: "LLMNode"` and your prompt
2. Receive a `workflow_id` in the response
3. Poll `GET /v1/node/status?workflow_id={id}` every 5 seconds until status is `completed`
4. Read the generated text from `output.text`

## Execute LLM

### Endpoint

`POST https://api.heygen.com/v1/node/execute`

### Request Fields

| Field | Type | Req | Description |
|-------|------|:---:|-------------|
| `node_type` | string | Y | Must be `"LLMNode"` |
| `input.prompt` | string | Y | The prompt or instruction for the LLM |
| `input.variables` | object | | Key-value pairs for template variable substitution in the prompt |

### curl

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "LLMNode",
    "input": {
      "prompt": "Write a 3-sentence product description for {{product_name}}",
      "variables": {"product_name": "HeyGen Video API"}
    }
  }'
```

### TypeScript

```typescript
interface LLMInput {
  prompt: string;
  variables?: Record<string, string>;
}

interface ExecuteResponse {
  data: {
    workflow_id: string;
    status: "submitted";
  };
}

async function runLLM(input: LLMInput): Promise<string> {
  const response = await fetch("https://api.heygen.com/v1/node/execute", {
    method: "POST",
    headers: {
      "X-Api-Key": process.env.HEYGEN_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      node_type: "LLMNode",
      input,
    }),
  });

  const json: ExecuteResponse = await response.json();
  return json.data.workflow_id;
}
```

### Python

```python
import requests
import os

def run_llm(prompt: str, variables: dict | None = None) -> str:
    payload = {
        "node_type": "LLMNode",
        "input": {"prompt": prompt},
    }

    if variables:
        payload["input"]["variables"] = variables

    response = requests.post(
        "https://api.heygen.com/v1/node/execute",
        headers={
            "X-Api-Key": os.environ["HEYGEN_API_KEY"],
            "Content-Type": "application/json",
        },
        json=payload,
    )

    data = response.json()
    return data["data"]["workflow_id"]
```

### Response Format

```json
{
  "data": {
    "workflow_id": "node-gw-e5f6g7h8",
    "status": "submitted"
  }
}
```

## Check Status

### Endpoint

`GET https://api.heygen.com/v1/node/status?workflow_id={workflow_id}`

### curl

```bash
curl -X GET "https://api.heygen.com/v1/node/status?workflow_id=node-gw-e5f6g7h8" \
  -H "X-Api-Key: $HEYGEN_API_KEY"
```

### Response Format (Completed)

```json
{
  "data": {
    "workflow_id": "node-gw-e5f6g7h8",
    "status": "completed",
    "output": {
      "text": "HeyGen Video API transforms your content into professional AI-powered videos in minutes..."
    }
  }
}
```

## Polling for Completion

```typescript
async function generateText(
  input: LLMInput,
  maxWaitMs = 120000,
  pollIntervalMs = 3000
): Promise<string> {
  const workflowId = await runLLM(input);
  console.log(`Submitted LLM job: ${workflowId}`);

  const startTime = Date.now();
  while (Date.now() - startTime < maxWaitMs) {
    const response = await fetch(
      `https://api.heygen.com/v1/node/status?workflow_id=${workflowId}`,
      { headers: { "X-Api-Key": process.env.HEYGEN_API_KEY! } }
    );
    const { data } = await response.json();

    switch (data.status) {
      case "completed":
        return data.output.text;
      case "failed":
        throw new Error(data.error?.message || "LLM generation failed");
      case "not_found":
        throw new Error("Workflow not found");
      default:
        await new Promise((r) => setTimeout(r, pollIntervalMs));
    }
  }

  throw new Error("LLM generation timed out");
}
```

## Usage Examples

### Simple Prompt

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "LLMNode",
    "input": {
      "prompt": "Write a short YouTube video description about AI avatars"
    }
  }'
```

### With Template Variables

```json
{
  "node_type": "LLMNode",
  "input": {
    "prompt": "Write a {{tone}} email subject line for {{topic}}",
    "variables": {
      "tone": "professional",
      "topic": "quarterly earnings report"
    }
  }
}
```

### Script Writing

```json
{
  "node_type": "LLMNode",
  "input": {
    "prompt": "Write a 60-second video script for a product demo. The product is {{product}}. Include an intro hook, 3 key features, and a call to action.",
    "variables": {
      "product": "AI-powered video translation"
    }
  }
}
```

## Best Practices

1. **Be specific in your prompts** — include desired length, tone, format, and audience
2. **Use template variables** for reusable prompts with dynamic content
3. **LLM responses are fast** — typically completes within 5-15 seconds; poll every 3 seconds
4. **Chain with other nodes** — use the generated text as input for TTS or video generation
5. **Check output.text** — the response contains the generated text as a plain string
