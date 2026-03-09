---
name: generate-image
description: |
  Generate images from text prompts using AI via HeyGen's Node Gateway. Use when: (1) Generating images from text descriptions, (2) Creating AI-generated visuals for videos or content, (3) Producing multiple image variations from a prompt, (4) Using reference images to guide generation, (5) Working with HeyGen's /v1/node/execute endpoint for image generation.
allowed-tools: mcp__heygen__*
metadata:
  openclaw:
    requires:
      env:
        - HEYGEN_API_KEY
    primaryEnv: HEYGEN_API_KEY
---

# Image Generation (HeyGen Node Gateway)

Generate images from text prompts using AI. Supports multiple image models, reference images for style guidance, and batch generation of multiple variations.

## Authentication

All requests require the `X-Api-Key` header. Set the `HEYGEN_API_KEY` environment variable.

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "GenerateImageNode", "input": {"prompt": "A professional headshot of a business executive, studio lighting"}}'
```

## Default Workflow

1. Call `POST /v1/node/execute` with `node_type: "GenerateImageNode"` and your prompt
2. Receive a `workflow_id` in the response
3. Poll `GET /v1/node/status?workflow_id={id}` every 5 seconds until status is `completed`
4. Download the generated images from the output URLs

## Execute Image Generation

### Endpoint

`POST https://api.heygen.com/v1/node/execute`

### Request Fields

| Field | Type | Req | Description |
|-------|------|:---:|-------------|
| `node_type` | string | Y | Must be `"GenerateImageNode"` |
| `input.prompt` | string | Y | Text description of the image to generate |
| `input.reference_images` | string[] | | URLs of reference images for style/content guidance |
| `input.num_generations` | number | | Number of image variations to generate (default: 1) |
| `input.model` | string | | Image model to use (default: `"gemini_image_generation"`) |

### curl

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "GenerateImageNode",
    "input": {
      "prompt": "A modern office background with soft lighting, suitable for a talking-head video",
      "num_generations": 3
    }
  }'
```

### TypeScript

```typescript
interface GenerateImageInput {
  prompt: string;
  reference_images?: string[];
  num_generations?: number;
  model?: string;
}

interface ExecuteResponse {
  data: {
    workflow_id: string;
    status: "submitted";
  };
}

async function generateImage(input: GenerateImageInput): Promise<string> {
  const response = await fetch("https://api.heygen.com/v1/node/execute", {
    method: "POST",
    headers: {
      "X-Api-Key": process.env.HEYGEN_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      node_type: "GenerateImageNode",
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

def generate_image(
    prompt: str,
    reference_images: list[str] | None = None,
    num_generations: int = 1,
    model: str | None = None,
) -> str:
    payload = {
        "node_type": "GenerateImageNode",
        "input": {
            "prompt": prompt,
            "num_generations": num_generations,
        },
    }

    if reference_images:
        payload["input"]["reference_images"] = reference_images
    if model:
        payload["input"]["model"] = model

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
    "workflow_id": "node-gw-i9j0k1l2",
    "status": "submitted"
  }
}
```

## Check Status

### Endpoint

`GET https://api.heygen.com/v1/node/status?workflow_id={workflow_id}`

### curl

```bash
curl -X GET "https://api.heygen.com/v1/node/status?workflow_id=node-gw-i9j0k1l2" \
  -H "X-Api-Key: $HEYGEN_API_KEY"
```

### Response Format (Completed)

```json
{
  "data": {
    "workflow_id": "node-gw-i9j0k1l2",
    "status": "completed",
    "output": {
      "images": [
        {"url": "https://resource.heygen.ai/generated/img1.png", "width": 1024, "height": 1024},
        {"url": "https://resource.heygen.ai/generated/img2.png", "width": 1024, "height": 1024}
      ]
    }
  }
}
```

## Polling for Completion

```typescript
interface GeneratedImage {
  url: string;
  width: number;
  height: number;
}

async function generateImageAndWait(
  input: GenerateImageInput,
  maxWaitMs = 300000,
  pollIntervalMs = 5000
): Promise<GeneratedImage[]> {
  const workflowId = await generateImage(input);
  console.log(`Submitted image generation: ${workflowId}`);

  const startTime = Date.now();
  while (Date.now() - startTime < maxWaitMs) {
    const response = await fetch(
      `https://api.heygen.com/v1/node/status?workflow_id=${workflowId}`,
      { headers: { "X-Api-Key": process.env.HEYGEN_API_KEY! } }
    );
    const { data } = await response.json();

    switch (data.status) {
      case "completed":
        return data.output.images;
      case "failed":
        throw new Error(data.error?.message || "Image generation failed");
      case "not_found":
        throw new Error("Workflow not found");
      default:
        await new Promise((r) => setTimeout(r, pollIntervalMs));
    }
  }

  throw new Error("Image generation timed out");
}
```

## Usage Examples

### Simple Prompt

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "GenerateImageNode",
    "input": {
      "prompt": "A clean, minimal product showcase on a white background"
    }
  }'
```

### Multiple Variations

```json
{
  "node_type": "GenerateImageNode",
  "input": {
    "prompt": "Professional corporate office background for video calls",
    "num_generations": 4
  }
}
```

### With Reference Images

```json
{
  "node_type": "GenerateImageNode",
  "input": {
    "prompt": "Similar style background but with warmer lighting and plants",
    "reference_images": ["https://example.com/reference-bg.png"]
  }
}
```

## Best Practices

1. **Be descriptive in prompts** — include lighting, style, composition, and mood details
2. **Use reference images** for style consistency across a series of generated images
3. **Generate multiple variations** with `num_generations` to pick the best result
4. **Image generation takes longer** than text — allow up to 60 seconds, poll every 5 seconds
5. **Output URLs are temporary** — download and store generated images promptly
