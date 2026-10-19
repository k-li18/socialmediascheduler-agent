# REST API and CLI reference

Base URL: `https://socialmediascheduler.cc/api/v1`
Auth: `Authorization: Bearer $SMS_API_KEY` (keys start with `sms_`). 120 requests a minute per key.
Full schema: `https://socialmediascheduler.cc/api/v1/openapi.json`

| Task | REST | CLI (`npx socialmediascheduler …`) | MCP tool |
|---|---|---|---|
| Check the key | `GET /me` | `whoami` | — |
| Connected accounts, by account set | `GET /social-accounts` → `{sets: [{id, name, active, accounts, not_connected}], sets_allowed}` | `accounts` | `list_social_accounts` |
| Link to connect accounts | `POST /social-accounts/connect-link` (`{"set_id"}` optional) | `connect-link [--set id]` | `get_connect_link` |
| Publish or schedule | `POST /posts` | `post "…" -p x,linkedin` | `create_post` |
| Check without publishing | `POST /posts/validate` | `post … --dry-run` | `validate_post` |
| List posts | `GET /posts?status=&limit=&offset=` | `posts` | `list_posts` |
| One post and its results | `GET /posts/{id}` | `post <id>` | `get_post` |
| Change caption or time | `PATCH /posts/{id}` | — | `update_post` |
| Cancel a scheduled post | `DELETE /posts/{id}` | `cancel <id>` | `cancel_post` |
| Delete a live post (Facebook, YouTube, X, LinkedIn, Threads) | `POST /posts/{id}/unpublish` (`{"platforms"}` optional) | `unpublish <id> [-p x] [--yes]` | `unpublish_post` |
| Upload media | `POST /media` (raw body + `Content-Type` + `?filename=`, or multipart `file`; 100 MB) | `upload <file>` | `upload_media` (base64, 5 MB) |
| Webhooks | `GET/POST /webhooks`, `GET/PATCH/DELETE /webhooks/{id}`, `POST /webhooks/{id}/test`, `POST /webhooks/{id}/rotate-secret`, `GET /webhooks/{id}/deliveries` | `webhooks [add|test|remove]` | `list_webhooks`, `create_webhook`, `delete_webhook`, `test_webhook` |

## Create a post

```bash
curl -X POST https://socialmediascheduler.cc/api/v1/posts \
  -H "Authorization: Bearer $SMS_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: launch-2026-11-02" \
  -d '{
    "caption": "We just launched!",
    "platforms": ["linkedin", "x"],
    "platform_captions": { "x": "Launched today 🚀" },
    "media_urls": ["https://example.com/launch.jpg"],
    "scheduled_at": "2026-11-02T09:00:00-05:00"
  }'
```

Body fields: `set_id` (which account set; default the first), `caption` (required), `platforms` (required), `platform_captions`, `media_urls` (HTTPS), `media_ids` (from upload), `story`, `youtube`, `scheduled_at`, `timezone`, `idempotency_key` (or the `Idempotency-Key` header), `dry_run`.

`youtube` is required when `platforms` includes `"youtube"`: `{"made_for_kids": false, "title": "Launch day", "privacy": "private"}`. `made_for_kids` is required (ask the user), `title` is optional (up to 100 characters; default the caption's first line), `privacy` is `public` (default), `unlisted` or `private`. CLI: `--not-made-for-kids --youtube-title "Launch day" --youtube-privacy private`.

201 means a new post; 200 with `"replayed": true` means the key matched an earlier post and nothing new was published. 422 `validation_failed` lists problems in `error.details.errors`.

## Post object

```json
{
  "id": "…",
  "status": "scheduled | publishing | published | partial | failed | cancelled | deleted",
  "type": "text | image | video | story",
  "caption": "…",
  "scheduled_at": "…", "published_at": "…", "created_at": "…",
  "source": "dashboard | api | mcp",
  "set_id": "…",
  "dashboard_url": "https://socialmediascheduler.cc/dashboard/posts?post=…",
  "results": [{ "platform": "x", "status": "published", "url": "https://x.com/…", "error": null }]
}
```

A result's `status` is `pending`, `publishing`, `published`, `failed` or `deleted`.

## Delete a published post

```bash
curl -X POST https://socialmediascheduler.cc/api/v1/posts/$POST_ID/unpublish \
  -H "Authorization: Bearer $SMS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "platforms": ["x", "linkedin"] }'
```

Omit `platforms` to delete it everywhere it is live. The answer is `{post, results: [{platform, ok, status, error}], deleted: [...]}`; platforms that cannot delete (Instagram, TikTok, …) appear in `results` with `ok: false` and the reason. When no platform deleted it the status is 422 `unpublish_failed` with the same `results` in `error.details`. A post that is not live anywhere answers 409 `invalid_state`.

## Verify a webhook delivery (Node)

```js
import { createHmac, timingSafeEqual } from 'node:crypto';

function verify(secret, headers, rawBody) {
  const id = headers['webhook-id'];
  const ts = headers['webhook-timestamp'];
  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) return false;
  const key = Buffer.from(secret.replace(/^whsec_/, ''), 'base64');
  const expected = 'v1,' + createHmac('sha256', key).update(`${id}.${ts}.${rawBody}`).digest('base64');
  return headers['webhook-signature'].split(' ').some(
    (sig) => sig.length === expected.length && timingSafeEqual(Buffer.from(sig), Buffer.from(expected))
  );
}
```

Any 2xx within 10 seconds counts as delivered; otherwise it is retried 8 more times over about 23 hours with the same `webhook-id`.
