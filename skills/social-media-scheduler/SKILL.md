---
name: social-media-scheduler
description: Publish or schedule social media posts to the user's connected Instagram, TikTok, YouTube, LinkedIn, Facebook, X, Threads, Pinterest, Bluesky and Google Business accounts through Social Media Scheduler (socialmediascheduler.cc), check how each platform's publish went, cancel or reschedule scheduled posts, delete posts that are already live, upload images and videos, and set up webhooks that fire when a post finishes. Use whenever the user wants to post, cross-post, schedule, queue or announce something on social media, asks what is scheduled or why a post failed, wants a published post taken down, or wants to automate social posting, even if they do not name the service.
---

# Social Media Scheduler

Social Media Scheduler publishes to the user's own social accounts. The user connects each network once on the website; you never see or handle their social logins.

## Pick a way in

Use the first one that works in this environment:

1. **MCP tools** (`list_social_accounts`, `create_post`, …). If they are already available, use them and skip the rest of this section. To add them in Claude Code: `claude mcp add --transport http social-media-scheduler https://socialmediascheduler.cc/api/mcp`, then sign in when asked.
2. **CLI**, if you can run shell commands: `npx socialmediascheduler <command> --json`. It needs `SMS_API_KEY` in the environment or a one-time `npx socialmediascheduler setup --key sms_…`.
3. **REST API**: `https://socialmediascheduler.cc/api/v1` with `Authorization: Bearer $SMS_API_KEY`. See [references/api.md](references/api.md).

API keys start with `sms_` and are created at Dashboard → Agents & API keys. Never print a key back to the user or write it into files they might commit.

## Workflow

1. **See what is connected.** `list_social_accounts` (CLI: `accounts`). Accounts come grouped in **account sets**: one set per brand or client, each with at most one account per platform. Only platforms connected in a set can be posted to from it. If the user has more than one set, ask which brand the post is for and pass its `set_id` to `create_post` and `get_connect_link` (CLI: `--set <id>`); without it the first set is used. A set with `active: false` is over the plan's limit and cannot post. If a platform the user wants is missing or `needs_reconnect` is true, call `get_connect_link` (CLI: `connect-link`), give the user the link, and wait for them to finish. You cannot connect accounts yourself.
2. **Draft with the user.** Keep each platform's limits in mind (below). Offer per-platform captions (`platform_captions`) when one caption does not fit everywhere, for example a short one for X.
3. **Media.** Public HTTPS links go in `media_urls`. Local files: `upload_media` (base64, up to 5 MB) or, with the CLI or REST, upload the file (up to 100 MB) and pass the returned `media_id` in `media_ids`. Several images make a carousel; a post carries either images or one video.
4. **YouTube.** When YouTube is a target, ask the user whether the video is made for kids and pass `youtube.made_for_kids` (`true` or `false`); the post is refused without it, and never guess it. Ask for a title too (`youtube.title`, up to 100 characters); otherwise the caption's first line is used. `youtube.privacy` is `public` by default; use `private` or `unlisted` when the user is testing or has not said it should go public. CLI: `--made-for-kids` or `--not-made-for-kids`, `--youtube-title`, `--youtube-privacy`.
5. **Check.** `validate_post` (CLI: `post … --dry-run`) runs every check without publishing. Fix errors; mention warnings to the user.
6. **Confirm, then publish.** Show the user the final caption(s), platforms, media and time (and for YouTube the title, privacy and kids answer), and get a clear yes. Then `create_post`. Always send an `idempotency_key` (any unique string, for example `launch-2026-11-02-linkedin`); if the call fails or times out, retry with the same key and you will never post twice.
7. **Report the outcome.** Publishing is asynchronous for most media. `get_post` refreshes the post and returns each platform's status, public link or error. A post ends `published` (all worked), `partial` (some failed) or `failed`.

## Scheduling

- Pass `scheduled_at` as ISO 8601 **with an offset** (`2026-11-02T09:00:00-05:00`), or a wall-clock time plus `timezone` (`America/New_York`). Ask the user's time zone if you do not know it; do not assume UTC.
- At least one minute ahead, at most one year ahead. Omit it to publish now.
- `update_post` changes the caption or time of a scheduled post; `cancel_post` cancels it. Neither works once the post is publishing or within about a minute of its time. Media and platforms of a scheduled post cannot change: cancel it and create a new one.
- Confirm with the user before cancelling.

## Deleting a published post

`unpublish_post` (CLI: `unpublish <id>`, REST: `POST /posts/{id}/unpublish`) deletes a post that is already live, from the `platforms` you list or from every platform it is live on. It cannot be undone: show the user the caption and the platforms it will be deleted from, and wait for a clear yes before calling.

- Works on Facebook, YouTube, X, LinkedIn and Threads. Instagram, TikTok and the other platforms have no delete API: their result comes back with `ok: false` and a message; tell the user to delete it in that app (the post's `url` points to it).
- Each platform reports its own result in `results`. If none could delete it, the call fails with `unpublish_failed` and the per-platform reasons.
- Deleted platforms show `status: "deleted"` on the post; once nothing is live any more the post itself becomes `deleted`.
- For a post that has not gone out yet, use `cancel_post` instead.

## Platform rules the checks enforce

| Platform | Needs media | Caption limit | Notes |
|---|---|---|---|
| Instagram | yes | 2,200 | up to 10 images; stories supported |
| TikTok | yes | 2,200 | |
| YouTube | video only | 5,000 | `youtube.made_for_kids` required; title up to 100 characters |
| Pinterest | yes | 500 | one image |
| LinkedIn | no | 3,000 | up to 9 images |
| Facebook | no | 63,206 | up to 10 images; stories supported |
| X | no | 280 | longer text is split into a thread; up to 4 images |
| Threads | no | 500 | longer text is split into a thread |
| Bluesky | no | 300 | longer text is split into a thread; up to 4 images |
| Google Business | no | 1,500 | |

Stories (`story: true`) work on Instagram and Facebook only and take exactly one image or video.

## Webhooks

For automations that should react when a post finishes, register a URL with `create_webhook` (CLI: `webhooks add <url>`, REST: `POST /webhooks`) instead of polling. Events: `post.published`, `post.partial`, `post.failed`; each carries the same post object `get_post` returns. The response includes a signing secret (`whsec_…`) that is shown once: tell the user to store it. Deliveries follow the Standard Webhooks format (`webhook-id`, `webhook-timestamp`, `webhook-signature` headers) and are retried for about a day. `test_webhook` sends a test event.

## Errors

Errors come back as `{"error": {"code", "message"}}`. The common ones:

- `validation_failed`: fix what `details.errors` lists and try again.
- `invalid_state` / `post_in_progress`: the post is no longer scheduled or is about to publish, or (for `unpublish_post`) it is not live anywhere.
- `unpublish_failed`: no platform deleted the post; `details.results` says why for each.
- `account_set`: the set does not exist, is over the plan's limit, or the plan has no sets yet; tell the user.
- `rate_limited`: 120 requests a minute per key; wait for `Retry-After`.
- `unauthorized`: the key is wrong or revoked, or the OAuth sign-in expired.
- `upstream_error`: the publishing service refused; tell the user the message.

## What this service does not do

Do not offer these, they do not exist here: engagement numbers (likes, views, followers), replying to comments or DMs, editing a post after it is published, deleting a published post from Instagram, TikTok or any platform other than Facebook, YouTube, X, LinkedIn and Threads, or posting to accounts the user has not connected.
