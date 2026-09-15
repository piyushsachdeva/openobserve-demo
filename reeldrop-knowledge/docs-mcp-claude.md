# ReelDrop Docs: Claude / MCP Integration

Sources: reeldrop.io/docs/connect-claude-to-reeldrop, schedule-reels-from-claude, mcp-security-plans-and-limits (fetched 2026-09-16)

## What it is
ReelDrop runs an MCP (Model Context Protocol) server so Claude Desktop, Claude Code, or any MCP-compatible client can operate a ReelDrop account by chat — schedule reels, edit the queue, pull analytics, manage DM automations — without a separate API key. Setup takes about 2 minutes.

## Plans
- **Included on Influencer ($9.99/mo) and Creator ($29/mo) plans.**
- Starter (free) users can connect the connector but get an upgrade prompt instead of tool access.
- Caption generations made via Claude count against the same monthly allowance as the app (unlimited on Creator).
- Scheduling itself is unlimited on every plan.

## Setup
1. Server URL: `https://mcp.reeldrop.io/mcp`
2. Claude Desktop: Settings → Connectors → Add custom connector → name it "ReelDrop", paste the URL.
   Claude Code: `claude mcp add --transport http reeldrop https://mcp.reeldrop.io/mcp`
3. A browser opens for OAuth sign-in with the user's own ReelDrop account — **no password or API key passes through the assistant.**
4. Verify: Claude Code → `claude mcp list` shows `reeldrop` as connected; Claude Desktop shows it in the tools menu. Test by asking "list my ReelDrop workspaces."
5. First scheduling request may ask for a timezone, which then saves to the workspace.

## What Claude can do once connected
- Schedule a single reel ("Schedule this reel for Tuesday evening" + a video link) — Claude proposes caption options and a cover, and confirms before booking.
- Bulk-schedule up to 15 reels in one request, distributed across saved posting slots.
- View, edit (caption/hashtags/timing/cover), or cancel already-scheduled content.
- Pull analytics: per-reel metrics, follower trends, comparisons across time periods.
- Create, modify, or attach DM automations (keyword triggers etc.) to reels before publishing.
- Always requires explicit user confirmation before actually publishing anything; scheduling requires a caption and a cover choice before it can be booked — no silent defaults.
- Each workspace is independent — the user must specify which workspace/Instagram account they mean if they have more than one.

## Security & limits
- OAuth tokens only — the ReelDrop password never passes through the assistant or the chat.
- Scope: can schedule, edit posts, generate captions, read analytics, manage DM automations. **Cannot** change billing, alter workspace settings, or connect/disconnect Instagram accounts.
- Revoke anytime via Claude's own connector settings, or `claude mcp remove reeldrop` in Claude Code.
- Per-account burst limits on individual tools to prevent abuse; Instagram's own platform cap (100 published posts per account per 24 hours) still applies underneath.
- ReelDrop does not use conversation content for model training. The user's Claude provider has its own separate privacy policy for conversation data.
- ReelDrop is listed on Smithery and the official MCP registry.
