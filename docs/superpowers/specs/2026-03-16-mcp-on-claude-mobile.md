# MCP on Claude Mobile — Design Spec
**Date:** 2026-03-16
**Status:** Draft

---

## Overview

This spec explores whether and how a Notion MCP server can be made available to Claude running on a mobile phone (iOS or Android), so that the Brain Dump capture workflow works end-to-end from the mobile app — without requiring a desktop session.

The short answer: **yes, it is possible**, but with constraints. Mobile supports only remote MCP servers (not local), and configuration happens on the web (not in the app itself). There are two viable paths depending on setup preferences.

---

## Goals

- Enable the Brain Dump capture workflow (`mcp__notion__API-*` tools) from Claude on mobile
- Avoid requiring a desktop machine to be running at capture time
- Keep the setup reproducible and documented so it survives app reinstalls or account changes

---

## Background: How MCP Works on Claude Mobile

### What is supported (as of July 2025)

Claude mobile apps (iOS and Android) support **remote MCP servers** only. A remote MCP server is an HTTPS endpoint that implements the MCP protocol — as opposed to a local stdio process running on your machine.

Once a remote MCP server is registered on claude.ai, it syncs automatically to the mobile app. There is no way to configure connectors directly inside the mobile app — configuration always goes through the web.

| Platform | Local MCP (stdio) | Remote MCP (HTTPS) |
|---|---|---|
| Claude Desktop | ✅ Supported | ✅ Supported |
| Claude Code CLI | ✅ Supported | ✅ Supported |
| Claude Web (claude.ai) | ✗ Not supported | ✅ Supported |
| Claude Mobile (iOS/Android) | ✗ Not supported | ✅ Supported |

### Plan requirement

Custom connectors (remote MCP) require a **Pro, Max, Team, or Enterprise** plan.

---

## Current Setup

The Brain Dump system currently uses a **local** Notion MCP server configured in Claude Code (or Claude Desktop). Tool calls use the `mcp__notion__API-*` prefix. This works on desktop but is unavailable in mobile sessions because:

1. The mobile app cannot run local processes
2. Local MCP servers are not exposed over HTTPS by default
3. The mobile app has no access to Claude Desktop's `claude_desktop_config.json`

---

## Options

### Option A — Use Claude Code Remote Control (Recommended for now)

**What it is:** Remote Control (released February 2026, research preview) syncs a running Claude Code terminal session with claude.ai and the Claude mobile app. Your local session — including all MCP integrations — stays running on your machine, and the mobile app connects to it as a remote interface.

**How it works for Brain Dump:**
1. Keep a Claude Code session running on your Mac (terminal or background process)
2. On mobile, open Claude → use Remote Control to connect to the active session
3. All `mcp__notion__API-*` tools from the local Notion MCP server are available in the mobile conversation

**Constraints:**
- Requires your Mac to be on and Claude Code running
- Remote Control is Pro/Max only; not available on Team or Enterprise plans
- Research preview — feature may change

**Best for:** Capture sessions where you're near your Mac (or comfortable leaving it running), and want zero additional infrastructure.

---

### Option B — Deploy the Notion MCP Server as a Remote HTTPS Endpoint

**What it is:** Host the Notion MCP server on a cloud service so it's reachable over HTTPS. Register the URL as a custom connector on claude.ai. Mobile (and web) then have access to Notion tools without any desktop dependency.

**Architecture:**
```
Claude Mobile
     │
     └──HTTPS──► Remote MCP Server (cloud-hosted)
                       │
                       └──Notion API──► Brain Dump Database
```

**Deployment options:**

| Option | Complexity | Cost | Notes |
|---|---|---|---|
| Cloudflare Workers | Low | Free tier available | Good for stateless MCP; no persistent process |
| Railway / Render | Low | ~$5/mo | Simple Node.js deploy; supports SSE transport |
| Google Cloud Run | Medium | Pay-per-request | Scales to zero; used in production MCP deployments |
| Self-hosted VPS | Medium | ~$5–10/mo | Full control; requires you to manage uptime |

**Transport:** Remote MCP servers use HTTP+SSE (Server-Sent Events) or the newer Streamable HTTP transport, not stdio.

**Authentication:** Claude's connector flow supports OAuth 2.1. For a personal Notion MCP server, a static bearer token (passed as an environment variable on the server) is simpler and sufficient.

**Steps to configure once deployed:**
1. Go to **claude.ai → Settings → Connectors → Add custom connector**
2. Enter the remote MCP server URL (e.g., `https://your-notion-mcp.yourdomain.com/mcp`)
3. Optionally add OAuth Client ID and Secret if using OAuth
4. Click **Add**
5. Settings sync automatically to Claude mobile app
6. In any conversation (web or mobile): tap **+** → **Connectors** → toggle the Notion connector on

**Constraints:**
- Requires initial deployment and maintenance of a cloud service
- Notion API key must be stored securely as an environment variable on the server
- Tool names may differ from the local MCP setup (`mcp__notion__API-*`) depending on the server implementation — may require updating `docs/notion-brain-dump/claude-instructions.md`

**Best for:** Fully mobile-independent capture — no desktop required, works anywhere.

---

### Option C — Use Notion's Pre-Built Connector (If Available)

Anthropic and Notion may offer a pre-built Notion connector that appears directly in claude.ai's connectors list without requiring a custom server URL. If this is available:

1. Go to **claude.ai → Settings → Connectors**
2. Look for Notion in the pre-built connectors list
3. Click **Connect** and complete the OAuth flow with your Notion account
4. Enable the connector in mobile conversations via **+** → **Connectors**

**Check first:** Verify whether the pre-built Notion connector exposes the same database query and page creation tools used in the Brain Dump workflow. If it only supports search or limited read access, Option B may still be needed for full write access.

---

## Recommended Path

| Scenario | Recommended Option |
|---|---|
| You want this working today with no new infra | **Option A** (Remote Control — Mac must be running) |
| You want fully mobile-independent capture | **Option B** (Deploy remote MCP server) |
| You want the simplest possible setup | **Option C** (Check for pre-built Notion connector first) |

Start with Option C (check if Notion is already a pre-built connector). If it covers the Brain Dump tool set, that's the fastest path. If not, use Option A while Option B is being set up.

---

## Impact on Existing Setup

Whichever option is chosen, the `docs/notion-brain-dump/claude-instructions.md` reference card may need one update: if the remote MCP server uses different tool name prefixes than the current `mcp__notion__API-*` names, update the MCP Tools table accordingly. Everything else (database ID, hierarchy, property formats, capture workflow) remains unchanged.

---

## Out of Scope

- Building a custom mobile app or Notion integration
- Automating capture without Claude (e.g., Zapier, webhooks)
- Multi-user or shared Brain Dump databases
- Offline capture (all options require network access to Notion)

---

## Success Criteria

- User can open Claude on phone, describe a thought (voice or text), and have it saved to the Notion Brain Dump database — without touching a desktop
- The capture workflow in `claude-instructions.md` executes correctly from a mobile session
- No manual steps required mid-capture (MCP tools are pre-authorized and active)
