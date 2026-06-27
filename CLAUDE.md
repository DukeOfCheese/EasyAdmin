# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

EasyAdmin is a FiveM/RedM server administration resource (v7.51) written in Lua (client/server) with a Node.js Discord bot. It manages in-game admin menus, bans, reports, permissions, and Discord integration.

## Build Commands

All bot build commands run from `src/`:

```bash
cd src && npm run build         # Install deps + compile bot JS to ../dist/
cd src && npm run update-deps   # Update dependencies to latest
```

The esbuild step (`src/build.js`) bundles `src/bot/*.js` and `src/bot/commands/*.js` into `dist/`. There are no tests and no linting commands configured.

## Architecture

### Lua Side (FiveM Resource)

```
client/admin_client.lua     — Initialization, network event registration, permission caching
client/gui_c.lua            — Entire NativeUI menu system (~70KB, all in-game UI)
server/admin_server.lua     — Core logic: player management, action execution, permission checks
server/banlist.lua          — Ban creation, identifier tracking, evasion detection, queries
server/storage.lua          — JSON persistence layer (SaveResourceFile/LoadResourceFile)
server/reports.lua          — Report system with cooldowns and dynamic thresholds
server/action_history.lua   — Audit trail for admin actions with configurable expiry
server/webhook.lua          — Discord webhook logging
shared/util_shared.lua      — Utilities, cooldowns, permission table (41 entries), identifiers
```

Communication is done entirely via **network events** (client ↔ server) and **exports/callbacks** (server ↔ bot).

### Bot Side (Node.js / Discord.js)

```
src/bot/bot.js              — Client init, slash command registration
src/bot/commands/           — 17 slash command files (ban, kick, unban, freeze, warn, etc.)
src/bot/chat_bridge.js      — Bidirectional game ↔ Discord chat sync
src/bot/server_status.js    — Live server status posting
src/bot/roles.js            — Discord role → ACE permission sync
```

Built output lands in `dist/` and is loaded by the FXServer manifest.

### Data Flow

1. Admin opens menu → client sends network event
2. Server validates ACE permissions → executes action
3. Storage writes to JSON files → webhook fires to Discord
4. Bot receives updates via FiveM exports; Discord commands trigger exports back to server

### Permission System

41 permissions in `shared/util_shared.lua` (`Permissions` table), grouped as `player.*`, `server.*`, `bot.*`, and specials (`immune`, `anon`). Permissions are checked via FXServer ACE and can be mapped to Discord roles.

### Storage Format

JSON files managed by `server/storage.lua`:
- **Bans**: `{id, name, identifiers[], moderator, reason, expires, type, timestamp}`
- **Actions**: `{action, identifiers[], reason, moderator, modIdentifiers[], timestamp}`
- **Notes**: `{text, identifiers[], author, authorIdentifiers[], timestamp}`

Ban evasion detection compares identifier arrays with a configurable minimum match count (`ea_minIdentifierMatches`, default 2).

### Plugin System

Extend EasyAdmin without modifying source files by registering a plugin table with `addPlugin()`. Plugins can add items to any menu (main, player, server, settings) or hook into menu open/close events. See `plugins/example_client.lua` and `plugins/example_server.lua` for templates.

### Configuration

40+ convars defined in `fxmanifest.lua`. Key ones:
- `ea_LanguageName` — language (default: `en`, options in `language/`)
- `ea_minIdentifierMatches` — ban evasion threshold
- `ea_botToken`, `ea_botLogChannel` — Discord bot
- `ea_screenshoturl` — screenshot upload endpoint
- `ea_maxWarnings`, `ea_warnAction` — warning system behavior
- `ea_actionHistoryExpiry` — audit log retention in days

### Localization

Language strings live in `language/<lang>.json` (de, en, es, fr, it, nl, pl). Always use the localization system rather than hardcoded strings.
