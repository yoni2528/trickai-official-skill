---
name: trickai-official-skill
description: Official reference for integrating with the Trick.AI API (trick-studio.com) — image and video generation. Trigger when the user asks to use, integrate, call, or build with the Trick.AI API, when they mention "trickai" or "trick-studio" in an integration context, or when they want to generate images/videos programmatically via Trick.AI (nano-banana, seedance-2.0 models).
---

# Trick.AI API Integration Skill

Use this skill whenever the user wants to integrate, call, or build with the Trick.AI API. It contains the full endpoint reference, request/response shapes, authentication, polling strategy, error handling, and credit semantics. When activated, generate code that is correct against this spec — do not guess endpoints or fields.

## When to use

Activate when the user:
- Asks to "use the trickai api" / "call trick-studio" / "integrate trickai"
- Wants to generate images or videos programmatically via Trick.AI
- Mentions Trick.AI models (`nano-banana`, `seedance-2.0`) in an integration context
- Asks about Trick.AI authentication, polling, credit costs, or webhooks

Do NOT activate for questions about the `modiglianiai` internal codebase itself — this skill is only for the **public Trick.AI API** consumed by third parties.

## Core facts (never guess these)

- **Base URL**: `https://trick-studio.com/api/v1`
- **Auth header**: `Authorization: Bearer sk-trk_YOUR_KEY`
- **API keys**: Created at `https://trick-studio.com/dashboard/api-keys`
- **Generation is async**: POST creates a job → poll GET until `status` is `completed` or `failed`
- **Credits**: Charged only on successful completion. Failed generations are free.
- **No hard rate limits**, but be reasonable — excessive usage may be throttled.

## Endpoints

### `GET /models`
Lists all available models with pricing.

Response:
```json
{
  "models": [
    { "id": "nano-banana", "name": "Nano Banana", "type": "image", "credit_cost": 0.5, "description": "..." }
  ]
}
```

### `GET /balance`
Returns remaining credits.
```json
{ "credits": 42.5 }
```

### `POST /generations`
Creates a generation job.

**Body fields:**
| Field | Type | Required | Notes |
|---|---|---|---|
| `model` | string | yes | e.g. `nano-banana`, `seedance-2.0` |
| `prompt` | string | yes | 1–5000 chars |
| `aspect_ratio` | string | no | `16:9`, `9:16`, `1:1` |
| `resolution` | string | no | `480p` or `720p` (video models) |
| `duration` | number | no | 4–15 seconds (Seedance models) |
| `image_urls` | string[] | no | Input image URLs for image-to-video |

**Response:**
```json
{ "id": "cmo8...", "status": "pending", "model": "nano-banana", "credit_cost": 0.5 }
```

### `GET /generations/:id`
Poll status. Response shape:
```json
{
  "id": "...",
  "type": "image",
  "model": "nano-banana",
  "prompt": "...",
  "status": "pending" | "processing" | "completed" | "failed",
  "credit_cost": 0.5,
  "url": null | "https://cdn.trickai.com/.../file.png",
  "error": null | "...",
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

### `GET /generations`
List past generations with pagination.
- Query params: `limit` (default 20, max 100), `offset` (default 0), `type` (`image` | `video`)
- Response: `{ data: [...], total, limit, offset }`

## Status values
- `pending` — submitted, waiting in queue
- `processing` — generation in progress
- `completed` — done; `url` contains the result
- `failed` — failed; `error` contains the reason

## Error codes
| HTTP | Code | Meaning |
|---|---|---|
| 401 | `invalid_api_key` | Missing/invalid/revoked key |
| 402 | `insufficient_credits` | Not enough credits |
| 400 | `invalid_model` | Model doesn't exist or is inactive |
| 422 | `validation_error` | Invalid request body |
| 404 | `not_found` | Generation not found |
| 500 | `generation_failed` | Internal server error |

## Polling strategy

Recommend **exponential backoff**: start at 2s, increase to 5–10s, cap at 15s. Give up after ~5 minutes with a timeout error.

Reference TypeScript implementation:

```typescript
async function waitForGeneration(id: string, apiKey: string): Promise<string> {
  const delays = [2000, 2000, 3000, 5000, 5000, 10000]; // then cap at 10s
  let attempt = 0;
  const deadline = Date.now() + 5 * 60 * 1000; // 5 min timeout

  while (Date.now() < deadline) {
    const res = await fetch(`https://trick-studio.com/api/v1/generations/${id}`, {
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    const job = await res.json();

    if (job.status === "completed") return job.url;
    if (job.status === "failed") throw new Error(`Generation failed: ${job.error}`);

    const delay = delays[Math.min(attempt++, delays.length - 1)];
    await new Promise((r) => setTimeout(r, delay));
  }
  throw new Error("Generation timed out after 5 minutes");
}
```

## Reference snippets

### Image generation (curl)
```bash
curl -X POST https://trick-studio.com/api/v1/generations \
  -H "Authorization: Bearer sk-trk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nano-banana",
    "prompt": "a red shoe on a white background, studio lighting"
  }'
```

### Video generation (curl)
```bash
curl -X POST https://trick-studio.com/api/v1/generations \
  -H "Authorization: Bearer sk-trk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "seedance-2.0",
    "prompt": "a cat walking on a beach at sunset",
    "aspect_ratio": "16:9",
    "resolution": "720p",
    "duration": 5
  }'
```

### Image-to-video (curl)
```bash
curl -X POST https://trick-studio.com/api/v1/generations \
  -H "Authorization: Bearer sk-trk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "seedance-2.0",
    "prompt": "the product slowly rotating",
    "image_urls": ["https://example.com/product.png"],
    "duration": 5
  }'
```

### Full end-to-end (TypeScript)
```typescript
const API_KEY = process.env.TRICKAI_API_KEY!;
const BASE_URL = "https://trick-studio.com/api/v1";

async function generate(model: string, prompt: string, extras: Record<string, unknown> = {}) {
  const create = await fetch(`${BASE_URL}/generations`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ model, prompt, ...extras }),
  });
  if (!create.ok) throw new Error(`Create failed: ${create.status} ${await create.text()}`);
  const { id } = await create.json();
  return waitForGeneration(id, API_KEY); // from polling section
}

// Image
const imageUrl = await generate("nano-banana", "a red shoe on white background");

// Video
const videoUrl = await generate("seedance-2.0", "a cat on a beach", {
  aspect_ratio: "16:9",
  resolution: "720p",
  duration: 5,
});
```

## Output format guidance

When generating integration code for the user:
1. **Always** use `Bearer` auth header exactly as shown; never invent alternate auth schemes
2. **Always** implement polling with backoff — do not tight-loop
3. **Always** handle the four status values, especially `failed` (surface `error` field)
4. **Never** hardcode API keys — read from env var (`TRICKAI_API_KEY` is the convention)
5. Prefer `fetch` over external HTTP libraries unless the user's stack requires otherwise
6. If the user wants to store results, note that `url` is a CDN link (public, suitable for direct use or re-upload)

## Common mistakes to avoid
- ❌ Assuming generation is synchronous — it isn't, you must poll
- ❌ Polling faster than 2s intervals — wasteful
- ❌ Charging the user's logic for failed jobs — the API doesn't charge for failures
- ❌ Using `api.trickai.com` or other guessed domains — the correct base is `trick-studio.com/api/v1`
- ❌ Sending credentials in query strings — always use the `Authorization` header
