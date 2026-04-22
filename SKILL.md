---
name: trickai-official-skill
description: Interactive onboarding and integration reference for the Trick.AI API (trick-studio.com). On activation, asks the user for their API key, fetches the list of available models, and shows what they can generate. Trigger when the user asks to use, integrate, call, connect to, or try the Trick.AI API, mentions "trickai" / "trick-studio" in an integration context, or wants to generate images/videos programmatically via Trick.AI.
---

# Trick.AI Official Skill

This skill onboards the user onto the Trick.AI API interactively. When activated, it runs a short conversation: collect the API key → show available models → offer to generate something. It also carries the full API reference so any follow-up code you write is correct.

## Activation flow — follow these steps in order

When this skill triggers, execute this sequence. Do not skip ahead.

### Step 1 — Greet and ask for the API key

Output this message verbatim (in the user's language — English or Hebrew):

> 👋 Trick.AI skill activated. To get started, I need your Trick.AI API key.
>
> 🔑 Please paste your API key (it starts with `sk-trk_`). You can create one at https://trick-studio.com/dashboard/api-keys
>
> 💡 Tip: if you've already set it as an environment variable named `TRICKAI_API_KEY`, just say "use env" and I'll use that instead.

Then **stop and wait** for the user's response. Do not proceed to Step 2 until you have a key.

### Step 2 — Validate and remember the key

Once the user provides a key:

1. **Store it for the rest of this conversation only** — do not write it to any file, do not commit it, do not include it in code snippets (use a placeholder like `YOUR_KEY` or `process.env.TRICKAI_API_KEY` in generated code).
2. **Validate format**: the key should match the pattern `sk-trk_[A-Za-z0-9_-]+`. If it doesn't, politely ask them to double-check.
3. **Test the key** by fetching the balance — this confirms it works before doing anything else:

```bash
export TRICKAI_API_KEY=<user's key>
curl -s -H "Authorization: Bearer $TRICKAI_API_KEY" https://trick-studio.com/api/v1/balance
```

   - If the response is `401` or `{"error": "invalid_api_key"}` → tell the user the key is invalid and go back to Step 1.
   - If valid, surface their balance: "✅ Key works. You have X credits."

### Step 3 — Fetch and show models

Call:

```bash
curl -s -H "Authorization: Bearer $TRICKAI_API_KEY" https://trick-studio.com/api/v1/models
```

Parse the response and present the models grouped by `type` (`image` vs `video`), like this:

```
🖼️  Image models
   • nano-banana (0.5 credits) — Fast product image generation
   • <other image models from response>

🎬  Video models
   • seedance-2.0 (N credits) — <description>
   • <other video models from response>
```

For each model, note the `credit_cost` so the user sees pricing upfront.

### Step 4 — Explain what they can do and ask what's next

Offer these three concrete paths:

1. **Generate an image** — "Tell me what to draw (e.g. 'a red shoe on a white background') and which model to use, and I'll call the API and wait for the result."
2. **Generate a video** — "Describe the scene, optionally give me an input image URL (for image-to-video), aspect ratio (16:9 / 9:16 / 1:1), resolution (480p / 720p), and duration (4–15s)."
3. **Write integration code** — "Tell me your language/stack (Node, Python, Go…) and I'll produce a ready-to-paste script against this API."

Wait for the user's choice.

### Step 5 — Execute the chosen path

For **generate image/video** paths, actually call the API in this session using Bash with the exported env var:

```bash
# Create the job
curl -s -X POST https://trick-studio.com/api/v1/generations \
  -H "Authorization: Bearer $TRICKAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "model": "<model>", "prompt": "<prompt>", ... }'
```

Capture the returned `id`, then poll with backoff (2s → 5s → 10s, cap at 15s, 5min timeout):

```bash
curl -s -H "Authorization: Bearer $TRICKAI_API_KEY" https://trick-studio.com/api/v1/generations/<id>
```

When `status` becomes `completed`, show the result `url` to the user. If `failed`, surface the `error` field.

For the **integration code** path, generate a script in their chosen language that follows the polling + error-handling patterns in the reference section below. Use `process.env.TRICKAI_API_KEY` (or equivalent) — never embed the actual key.

## Security rules (strict)

- ❌ **Never write the API key to any file** on the user's disk unless they explicitly ask.
- ❌ **Never include the raw key in any code snippet** you show — always use env-var references.
- ❌ **Never log the key** in echo commands, comments, or debug output.
- ✅ When calling curl in Bash, pass the key via `-H "Authorization: Bearer $TRICKAI_API_KEY"` after `export TRICKAI_API_KEY=...` so the literal value isn't repeated in your output.
- ✅ If the user wants to persist the key, suggest they add `export TRICKAI_API_KEY=sk-trk_...` to `~/.zshrc` themselves — do not do it for them.

## API reference (use these exactly — never guess)

### Base URL
`https://trick-studio.com/api/v1`

### Auth
`Authorization: Bearer sk-trk_YOUR_KEY`

### Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/models` | List available models with pricing |
| GET | `/balance` | Get remaining credits |
| POST | `/generations` | Create an image/video job |
| GET | `/generations/:id` | Poll a job's status |
| GET | `/generations?limit=N&offset=M&type=image\|video` | List past jobs |

### `POST /generations` body
| Field | Type | Required | Notes |
|---|---|---|---|
| `model` | string | ✅ | e.g. `nano-banana`, `seedance-2.0` |
| `prompt` | string | ✅ | 1–5000 chars |
| `aspect_ratio` | string | ⬜ | `16:9`, `9:16`, `1:1` |
| `resolution` | string | ⬜ | `480p` or `720p` (video) |
| `duration` | number | ⬜ | 4–15 seconds (Seedance) |
| `image_urls` | string[] | ⬜ | Input images for image-to-video |

### Status values
`pending` → `processing` → `completed` (returns `url`) or `failed` (returns `error`)

### Error codes
| HTTP | Code | Meaning |
|---|---|---|
| 401 | `invalid_api_key` | Missing/invalid/revoked key |
| 402 | `insufficient_credits` | Not enough credits |
| 400 | `invalid_model` | Model doesn't exist or is inactive |
| 422 | `validation_error` | Invalid request body |
| 404 | `not_found` | Generation not found |
| 500 | `generation_failed` | Internal server error |

### Credits
Charged only on successful completion. Failed generations are free. Check balance with `GET /balance`.

## Reference polling implementation (TypeScript)

```typescript
async function waitForGeneration(id: string, apiKey: string): Promise<string> {
  const delays = [2000, 2000, 3000, 5000, 5000, 10000]; // then cap at 10s
  let attempt = 0;
  const deadline = Date.now() + 5 * 60 * 1000;

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

## Full end-to-end reference (TypeScript)

```typescript
const API_KEY = process.env.TRICKAI_API_KEY!;
const BASE = "https://trick-studio.com/api/v1";

async function generate(model: string, prompt: string, extras: Record<string, unknown> = {}) {
  const res = await fetch(`${BASE}/generations`, {
    method: "POST",
    headers: { Authorization: `Bearer ${API_KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({ model, prompt, ...extras }),
  });
  if (!res.ok) throw new Error(`Create failed: ${res.status} ${await res.text()}`);
  const { id } = await res.json();
  return waitForGeneration(id, API_KEY);
}

// Usage
const imageUrl = await generate("nano-banana", "a red shoe on white");
const videoUrl = await generate("seedance-2.0", "a cat on a beach", {
  aspect_ratio: "16:9", resolution: "720p", duration: 5,
});
```

## Common mistakes to avoid

- ❌ Skipping Step 1 — always ask for the key first, even if the user dives straight into "generate me a cat video"
- ❌ Hardcoding the key into code snippets
- ❌ Tight-looping without backoff when polling
- ❌ Assuming the generation is synchronous
- ❌ Using wrong domain like `api.trickai.com` — it's `trick-studio.com/api/v1`
- ❌ Charging logic for failed jobs — the API doesn't charge for failures
