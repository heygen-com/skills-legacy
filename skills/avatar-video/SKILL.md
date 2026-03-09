---
name: avatar-video
description: |
  Generate an AI avatar video from audio using HeyGen's Node Gateway. Use when: (1) Creating a talking-head video with an AI avatar, (2) Driving an avatar with audio to produce a video, (3) Generating avatar video from speech audio, (4) Working with HeyGen's /v1/node/execute endpoint for avatar inference.
allowed-tools: mcp__heygen__*
metadata:
  openclaw:
    requires:
      env:
        - HEYGEN_API_KEY
    primaryEnv: HEYGEN_API_KEY
---

# Avatar Video Generation (HeyGen Node Gateway)

Generate an AI avatar video driven by audio. Provide an avatar ID and one or more audio segments to produce a talking-head video.

## Authentication

All requests require the `X-Api-Key` header. Set the `HEYGEN_API_KEY` environment variable.

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "AvatarInferenceNode", "input": {"avatar": {"avatar_id": "YOUR_AVATAR_ID"}, "audio_list": [{"audio_url": "https://example.com/speech.mp3"}]}}'
```

## Default Workflow

1. Call `POST /v1/node/execute` with `node_type: "AvatarInferenceNode"`, an avatar ID, and audio segments
2. Receive a `workflow_id` in the response
3. Poll `GET /v1/node/status?workflow_id={id}` every 10 seconds until status is `completed`
4. Use the returned `video_url` from the output

## Execute Avatar Video

### Endpoint

`POST https://api.heygen.com/v1/node/execute`

### Request Fields

| Field | Type | Req | Description |
|-------|------|:---:|-------------|
| `node_type` | string | Y | Must be `"AvatarInferenceNode"` |
| `input.avatar` | object | Y | `{"avatar_id": "..."}` — the avatar to animate |
| `input.audio_list` | array | Y | List of audio objects: `[{"audio_url": "https://..."}]` |
| `input.audio_delay_list` | number[] | | Delay in seconds before each audio segment (default: no delays) |

### curl

```bash
curl -X POST "https://api.heygen.com/v1/node/execute" \
  -H "X-Api-Key: $HEYGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "AvatarInferenceNode",
    "input": {
      "avatar": {"avatar_id": "Angela-inblackskirt-20220820"},
      "audio_list": [
        {"audio_url": "https://example.com/intro.mp3"},
        {"audio_url": "https://example.com/main-content.mp3"}
      ],
      "audio_delay_list": [0, 1.5]
    }
  }'
```

### TypeScript

```typescript
interface AvatarVideoInput {
  avatar: { avatar_id: string };
  audio_list: Array<{ audio_url: string }>;
  audio_delay_list?: number[];
}

async function generateAvatarVideo(input: AvatarVideoInput): Promise<string> {
  const response = await fetch("https://api.heygen.com/v1/node/execute", {
    method: "POST",
    headers: {
      "X-Api-Key": process.env.HEYGEN_API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      node_type: "AvatarInferenceNode",
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

def generate_avatar_video(
    avatar_id: str,
    audio_urls: list[str],
    audio_delays: list[float] | None = None,
) -> str:
    input_data = {
        "avatar": {"avatar_id": avatar_id},
        "audio_list": [{"audio_url": url} for url in audio_urls],
    }

    if audio_delays:
        input_data["audio_delay_list"] = audio_delays

    response = requests.post(
        "https://api.heygen.com/v1/node/execute",
        headers={
            "X-Api-Key": os.environ["HEYGEN_API_KEY"],
            "Content-Type": "application/json",
        },
        json={"node_type": "AvatarInferenceNode", "input": input_data},
    )

    return response.json()["data"]["workflow_id"]
```

### Response Format

```json
{
  "data": {
    "workflow_id": "node-gw-a1v2t3r4",
    "status": "submitted"
  }
}
```

## Check Status

### Endpoint

`GET https://api.heygen.com/v1/node/status?workflow_id={workflow_id}`

### curl

```bash
curl -X GET "https://api.heygen.com/v1/node/status?workflow_id=node-gw-a1v2t3r4" \
  -H "X-Api-Key: $HEYGEN_API_KEY"
```

### Response Format (Completed)

```json
{
  "data": {
    "workflow_id": "node-gw-a1v2t3r4",
    "status": "completed",
    "output": {
      "video": {
        "video_url": "https://resource.heygen.ai/avatar/output.mp4",
        "duration": 45.2
      }
    }
  }
}
```

## Polling for Completion

```typescript
async function generateAvatarVideoAndWait(
  input: AvatarVideoInput,
  maxWaitMs = 600000,
  pollIntervalMs = 10000
): Promise<{ video_url: string; duration: number }> {
  const workflowId = await generateAvatarVideo(input);
  console.log(`Submitted avatar video: ${workflowId}`);

  const startTime = Date.now();
  while (Date.now() - startTime < maxWaitMs) {
    const response = await fetch(
      `https://api.heygen.com/v1/node/status?workflow_id=${workflowId}`,
      { headers: { "X-Api-Key": process.env.HEYGEN_API_KEY! } }
    );
    const { data } = await response.json();

    switch (data.status) {
      case "completed":
        return data.output.video;
      case "failed":
        throw new Error(data.error?.message || "Avatar video generation failed");
      case "not_found":
        throw new Error("Workflow not found");
      default:
        await new Promise((r) => setTimeout(r, pollIntervalMs));
    }
  }

  throw new Error("Avatar video generation timed out");
}
```

## Usage Examples

### Single Audio Segment

```json
{
  "node_type": "AvatarInferenceNode",
  "input": {
    "avatar": {"avatar_id": "Angela-inblackskirt-20220820"},
    "audio_list": [
      {"audio_url": "https://example.com/narration.mp3"}
    ]
  }
}
```

### Multi-Segment with Delays

```json
{
  "node_type": "AvatarInferenceNode",
  "input": {
    "avatar": {"avatar_id": "Angela-inblackskirt-20220820"},
    "audio_list": [
      {"audio_url": "https://example.com/greeting.mp3"},
      {"audio_url": "https://example.com/explanation.mp3"},
      {"audio_url": "https://example.com/outro.mp3"}
    ],
    "audio_delay_list": [0, 2.0, 1.0]
  }
}
```

## Best Practices

1. **Get avatar IDs from `GET /v2/avatars`** — use the HeyGen API to list available avatars
2. **Use clear, high-quality audio** — audio quality directly impacts lip-sync accuracy
3. **Avatar video is the slowest node** — expect 1-5 minutes depending on audio length; poll every 10 seconds
4. **Chain with TTS first** — generate audio with text-to-speech, then drive the avatar with that audio
5. **Use `audio_delay_list`** for natural pacing — add pauses between audio segments
6. **Max timeout of 10 minutes** — very long audio segments may take longer; set generous polling timeouts
