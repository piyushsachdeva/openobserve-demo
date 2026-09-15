# ReelDrop Docs: DM Automation

Sources: reeldrop.io/docs/create-your-first-dm-automation, comment-to-dm-setup, follow-gates-and-follow-verification,
story-reply-triggers, dm-keyword-triggers, timed-follow-ups, dm-send-limits-and-account-safety,
dm-automation-limits-by-plan (fetched 2026-09-16)

## What DM automation is

Automated Instagram DMs triggered by an interaction. Four trigger types:
1. **Comment on post/reel** ("comment-to-DM") — someone comments a keyword on a post/reel/carousel.
2. **Incoming DM with keyword match** — someone DMs the account and it contains a keyword.
3. **Story reply** — someone replies to a story.
4. **DM reply by AI** — Creator plan only.

Requires a Business/Creator Instagram account (personal accounts have no messaging API access).

## Setup (3 steps)
1. **Name & trigger** — pick one of the four trigger types above; for keyword triggers choose match mode: Contains keyword (default), Exact match, or Any.
2. **Actions** — up to three messages in sequence: an opening message with a button (default on), an optional follow-gate request (paid plans only), and the main DM with up to 3 link buttons (20 chars max per label, must be https://).
3. **Optional follow-up** — one delayed message (30 min – 24 hours) sent only if the person hasn't replied yet; automatically cancelled if they reply first. No chaining/branching — exactly one follow-up per automation, by design (avoids multi-step drip campaigns that risk account restrictions).

## Attaching automation to a scheduled post
- Build the automation first with its post target set to "Attach while posting" (keeps it dormant until attached).
- In the post composer, use the "Attach Automation" card to bind an active automation to that specific Reel/post/carousel before it publishes — no gap where the post is live but the automation isn't.
- Stories use the same attach flow but only show story-reply automations (stories have no comments).
- **Retrigger**: replay automation against a post's *existing* comments (Instagram doesn't webhook old comments) — up to 50 comments across 5 posts.
- One automation can attach to multiple posts (target = "All reels & posts" or a specific list); deleting the attached post doesn't delete the automation, it just stops receiving events.

## Follow gates
- Requires following the account before the link is delivered, verified via Instagram Graph API's `is_user_follow_business` field.
- Three outcomes: confirmed following → link sends immediately; confirmed not following → re-prompt, link withheld; **no data available → "trust mode," link sends anyway** (avoids stranding a real follower behind a check that failed on Meta's side).
- Verification happens once, at delivery — an unfollow afterward doesn't retract the message already sent.
- Works on comment, story-reply, and incoming-DM triggers; not available on AI-generated DM replies.
- No per-person reporting on who was gated vs. got through.

## Story-reply specifics
- Only question-sticker replies or text replies trigger automations — poll stickers and sliders do not.
- Retrigger is not supported for story automations.
- A single automation can't handle both comment and story-reply triggers.
- Follow gates are usually ineffective here since story repliers are typically already followers.

## DM keyword triggers
- Turns a frequently-asked question into a self-serve automated answer.
- Requires a paid plan (Influencer $9.99/mo minimum).
- No retrigger capability on past DMs; testing requires a separate Instagram account.

## Limits and account safety
| Constraint | Limit |
|---|---|
| Active automations | 5 on Starter (free); unlimited on Influencer/Creator |
| Link buttons per message | 3 max |
| Retrigger scope | 50 comments across 5 posts |
| Follow-ups per automation | 1 (no chaining) |
| Instagram API publishing rate | 100 posts per account per 24 hours (Instagram's own limit, not ReelDrop's) |

- ReelDrop does **not** cap total daily DM volume itself — Instagram's own messaging rules apply server-side.
- Instagram's messaging window: a business can message someone for 24 hours after that person messages the business. Comment-to-DM is Meta's documented exception, allowing outreach to someone who engaged (commented) but never DM'd first.
- Safety mechanisms: webhook verification, self-event filtering, idempotent logging, URL validation — prevents loops, duplicate sends, and malformed requests.
- ReelDrop uses the official Graph API with granted permissions only — never asks for the Instagram password, never simulates a browser or scrapes.

## Automation limits by plan
- **Starter (Free)**: 5 active DM automations (comment triggers), uncapped DM volume through them.
- **Influencer ($9.99/mo or $79/yr)**: unlimited active automations, all trigger types (story-reply, DM-keyword, follow-gate).
- **Creator ($29/mo, $149/yr, or $249 lifetime)**: unlimited automations + AI-generated DM replies + up to 5 workspaces.
- Paused automations don't count against the limit — only active ones consume the quota.
- Limits apply per workspace (Starter/Influencer: 1 workspace; Creator: up to 5).
