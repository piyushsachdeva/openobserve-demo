# ReelDrop Features Overview

Source: https://www.reeldrop.io/features (fetched 2026-09-16)

ReelDrop is an all-in-one Instagram management platform that consolidates several creator tools into one app: a scheduler, an AI carousel/caption generator, DM automation, analytics, and content calendar — aimed at replacing multiple separate subscriptions.

## Core capabilities

- **Scheduling**: Posts and reels publish automatically through Instagram's official Graph API — server-side, so publishing works whether or not the user's phone is on, with no manual confirmation step required.
- **AI Carousels**: Turns a topic into a designed 10-slide carousel, with 30+ themes to choose from.
- **Caption & hashtag generation**: Produces caption options (multiple tone variants) with character counts, plus hashtag suggestions.
- **DM automation**: Comment-to-DM (auto-reply to comments containing a keyword), story-reply automation, DM keyword triggers, and follow gates.
- **Content calendar**: Visual calendar for planning and viewing scheduled content across the workspace.
- **Analytics**: Native Instagram analytics inside ReelDrop, including backfilled data from existing native posts.
- **MCP (Model Context Protocol) integration**: ReelDrop runs an MCP server so Claude Desktop, Claude Code, or any MCP-compatible client can schedule posts, edit the queue, pull analytics, and manage DM automations directly from a chat conversation — no separate API key needed, authenticated via OAuth with the user's own ReelDrop account.
- **Free tools**: ~20 free utilities (caption length checker, hashtag generators, engagement rate calculator, aspect ratio checker, etc.), usable without creating an account.

See `pricing.md` for exactly which features are gated to which plan, and the `docs-*.md` files for step-by-step setup guides on DM automation, scheduling, and MCP.
