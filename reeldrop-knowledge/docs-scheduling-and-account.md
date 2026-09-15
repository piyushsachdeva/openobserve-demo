# ReelDrop Docs: Connecting Instagram & Scheduling Reels

Sources: reeldrop.io/docs/connect-your-instagram-account, reeldrop.io/docs/schedule-your-first-reel (fetched 2026-09-16)

## Connecting an Instagram account

- Requires an Instagram **Business or Creator** account — personal accounts have no API access. Converting is free in the Instagram app (Settings → Account type and tools).
- Connect via Settings → Connections → "Connect Instagram". A popup shows Instagram's own authorization screen; the user signs in with the professional account's credentials even if already logged in elsewhere.
- Instagram asks approval for: basic account info, publishing, comment management, messaging, and insights — each maps to a specific ReelDrop feature.
- On success, the workspace adopts the connected account's name (unless already customized) and ReelDrop backfills existing native Instagram posts into analytics.
- **Tokens expire after 60 days.** ReelDrop checks token health daily; if it expires or access is revoked from Instagram's side, ReelDrop flags the connection, emails the user, and prompts a reconnect.
- **One Instagram account per workspace.** Connecting a second account requires a second workspace (Creator plan supports up to 5 workspaces/accounts).
- Common issue: a login loop usually means the account still needs a password set directly in Instagram — the popup uses Instagram's own credential form.
- ReelDrop never asks for or stores the Instagram password itself; the whole flow runs through Instagram's official OAuth screen.

## Scheduling a reel

- Free plan includes: unlimited reel scheduling and posting, 5 active DM automations (comment triggers, uncapped DM volume), 3 AI caption generations/month, 3 trial reels/month.
- Flow: upload the video → pick a cover → write or AI-generate the caption → optionally attach a DM automation → set the publish time → ReelDrop auto-publishes.
- Publishing is **server-side** through the official Instagram API — the reel goes out even if the phone is off, nothing to manually confirm.
- Requires a Business/Creator account and a 9:16-aspect-ratio video (recommended safe format).
- Advanced: first-comment attachment (Influencer/Creator plans), trial reel publishing to non-followers first, carousel and story support, timezone-based scheduling from account settings.
- Failed publishes trigger an in-app notification and an email with the reason. Slow transcoding shows as "delayed" and auto-retries once Instagram finishes processing.
