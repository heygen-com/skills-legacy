---
name: content-moderation
description: |
  Check text, image, audio, or video content against moderation policies using HeyGen's Node Gateway. Use when: (1) Moderating user-submitted content before publishing, (2) Checking text or media for policy violations, (3) Validating content safety before video generation, (4) Working with HeyGen's /v1/node/execute endpoint for content moderation.
allowed-tools: mcp__heygen__*
metadata:
  openclaw:
    requires:
      env:
        - HEYGEN_API_KEY
    primaryEnv: HEYGEN_API_KEY
---

# Content Moderation (HeyGen Node Gateway)

Check text, image, audio, or video content against moderation policies. Returns accept/reject decisions with confidence scores and reasons.

## Authentication

All requests require the `X-Api-Key` header. Set the `HEYGEN_API_KEY` environment variable.

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "ModerationNode", "input": {"text": "Content to check for policy compliance"}}'
```

## Default Workflow

1. Call `POST /v1/node/execute` with `node_type: "ModerationNode"` and content to check
2. Receive a `workflow_id` in the response
3. Poll `GET /v1/node/status?workflow_id={id}` every 5 seconds until status is `completed`
4. Check `output.evaluation` for the moderation decision and `output.reason` for details

## Execute Moderation

### Endpoint

`POST https://api.heygen.com/v1/node/execute`

### Request Fields

| Field | Type | Req | Description |
|-------|------|:---:|-------------|
| `node_type` | string | Y | Must be `"ModerationNode"` |
| `input.text` | string | | Text content to moderate |
| `input.asset` | object | | Media asset to moderate — audio, video, or image object with URL |
| `input.source_type` | string | | Content source context (default: `"generic"`) |
| `input.moderation_types` | string[] | | Types of moderation to apply (default: `["text", "media"]`) |
| `input.raise_on_reject` | boolean | | Whether to raise an error on rejection (default: true) |

### curl

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "ModerationNode",
    "input": {
      "text": "Professional product description for our new software release",
      "moderation_types": ["text"]
    }
  }'
```

### TypeScript

```typescript
interface ModerationInput {
  text?: string;
  asset?: { audio_url: string } | { video_url: string } | { image_url: string };
  source_type?: string;
  moderation_types?: string[];
  raise_on_reject?: boolean;
}

async function moderate(input: ModerationInput): Promise<string> {
  const response = await fetch("https://api.heygen.com/v1/node/execute", {
    method: "POST",
    headers: {
      "X-Api-Key": process.env.HEYGEN_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ node_type: "ModerationNode", input }),
  });

  const json = await response.json();
  return json.data.workflow_id;
}
```

### Python

```python
import requests
import os

def moderate_content(
    text: str | None = None,
    asset_url: str | None = None,
    asset_type: str = "image",
    moderation_types: list[str] | None = None,
) -> str:
    input_data: dict = {}
    if text:
        input_data["text"] = text
    if asset_url:
        input_data["asset"] = {f"{asset_type}_url": asset_url}
    if moderation_types:
        input_data["moderation_types"] = moderation_types
    input_data["raise_on_reject"] = False

    response = requests.post(
        "https://api.heygen.com/v1/node/execute",
        headers={
            "X-Api-Key": os.environ["HEYGEN_API_KEY"],
            "Content-Type": "application/json",
        },
        json={"node_type": "ModerationNode", "input": input_data},
    )

    return response.json()["data"]["workflow_id"]
```

### Response Format (Completed)

```json
{
  "data": {
    "workflow_id": "node-gw-m0d1e2r3",
    "status": "completed",
    "output": {
      "evaluation": "accept",
      "reason": "Content meets all policy requirements",
      "confidence": 0.98,
      "passed": true
    }
  }
}
```

## Best Practices

1. **Set `raise_on_reject: false`** to get the moderation result instead of an error on rejection
2. **Check both text and media** when moderating multimodal content
3. **Use before video generation** to catch policy issues before spending compute on generation
4. **Moderation is fast** — typically completes within 5-15 seconds
5. **Check `confidence`** — lower confidence scores may warrant manual review
