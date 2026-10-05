# NFL MCP Server — Design

**Date:** 2026-10-05
**Status:** Approved in brainstorming, pending spec review

## Purpose

A small, polished Model Context Protocol server for a portfolio: something a
potential employer can clone, run in one command and read in about ten
minutes. It answers questions about NFL teams (record, standing, schedule,
season stats) from ESPN's public JSON endpoints.

It is a standalone public project. It shares no code, data, endpoints or
naming with any employer's systems.

## Success criteria

- `npm install && npm run build` then adding the server to Claude Desktop or
  Claude Code works with no API keys or other setup.
- "How are the Eagles doing?" and "Preview the Chiefs' next game" give
  correct, current answers.
- `npm test` passes offline (no network calls).
- CI on GitHub runs lint, typecheck, tests and a Docker build on every push.
- The README tells a reviewer what the project is, how to run it, and why it
  is built the way it is.

## Stack

- TypeScript (strict), Node 20+ (developed on Node 24), ESM.
- `@modelcontextprotocol/sdk` 1.x (1.32.1 current), `zod` for input schemas.
- Vitest for tests, ESLint + Prettier for lint and formatting.
- No runtime dependencies beyond the SDK and zod. HTTP uses the built-in `fetch`.

## Data source

ESPN's public, keyless, **unofficial and undocumented** site API. All of it is
reached through `src/espn/client.ts`, so a change on ESPN's side or a switch of
provider touches one file. The README states plainly that it is unofficial.

Endpoints in use (all verified working 2026-10-05):

| Use | URL |
|---|---|
| All teams | `https://site.api.espn.com/apis/site/v2/sports/football/nfl/teams` |
| Team detail (record, standing, next game) | `.../nfl/teams/{id}` |
| Team schedule | `.../nfl/teams/{id}/schedule?season={year}` |
| Team season stats | `.../nfl/teams/{id}/statistics` |
| Standings | `https://site.api.espn.com/apis/v2/sports/football/nfl/standings` |

## Architecture

```
src/
  espn/client.ts      fetch wrapper: base URL, timeout, typed JSON, EspnError
  espn/cache.ts       small TTL cache keyed by URL
  espn/types.ts       only the response fields we read
  espn/teams.ts       resolveTeam(): "PHI" | "eagles" | "philadelphia" | "21" → team(s)
  tools/              search-teams.ts, team-overview.ts, team-schedule.ts, team-stats.ts
  resources/          teams.ts, standings.ts
  prompts/            game-preview.ts
  server.ts           createServer(deps): McpServer with everything registered
  stdio.ts            entry: stdio transport (default)
  http.ts             entry: Streamable HTTP, POST /mcp, GET /health
test/
  fixtures/           recorded ESPN responses (JSON)
  *.test.ts
```

**Boundaries:**
- `espn/` knows HTTP and ESPN's response shapes, and nothing about MCP.
- `tools/`, `resources/` and `prompts/` each export a `register(server, deps)`
  function. They depend on an `EspnClient` interface, not on `fetch`.
- `server.ts` wires the pieces together and knows no transport. Both entry
  files and the integration test call `createServer()`.

## Tools

Every tool takes `team` as a string: an abbreviation, nickname, city or ESPN
id. `resolveTeam` matches case-insensitively. Exactly one match → proceed.
Several (e.g. "new york" → Giants, Jets; "los angeles" → Rams, Chargers) → tool
error listing the candidates. None → tool error suggesting `search_teams`.

| Tool | Input | Output |
|---|---|---|
| `search_teams` | `query` | Matching teams: id, abbreviation, name, conference/division if available |
| `get_team_overview` | `team` | Record, standing summary (e.g. "2nd in NFC East"), next game (opponent, date, home/away) |
| `get_team_schedule` | `team`, `season?` (defaults to current) | Each game: week, date, opponent, home/away, final score + W/L, or "scheduled" |
| `get_team_stats` | `team`, `category?` (`passing`, `rushing`, `receiving`, `defense`, …) | Season stats grouped by category; filtered when a category is given |

Each result returns a concise text summary in `content` plus `structuredContent`
matching a declared `outputSchema`.

## Resources

- `nfl://teams`: all 32 teams as JSON (id, abbreviation, name, location).
- `nfl://standings`: current standings by conference and division (team, W-L-T,
  win %).

## Prompt

`game_preview` with argument `team`. It returns a message telling the model to
call `get_team_overview` to find the next opponent, then `get_team_overview`
and `get_team_stats` for both teams, then write a short preview comparing
records, offense and defense.

## Caching

An in-memory TTL cache in front of the client, keyed by URL: 10 minutes for
schedules, stats and standings, 24 hours for the team list. It holds at most a
few hundred entries and has no eviction beyond the TTL; the data set is tiny.

## Error handling

- `EspnError` carries an HTTP status or a timeout. The default timeout is 10 s
  via `AbortSignal.timeout`.
- Tools catch everything and return `isError: true` with an actionable
  message. A tool never throws into the SDK, so the server process never
  crashes on bad input or ESPN downtime.
- Zod rejects invalid input before a handler runs (e.g. a season outside 2002
  to next year).
- Missing optional fields in ESPN responses (e.g. no next game in the
  offseason) produce "none scheduled", not a crash.

## Transports

- **stdio** (`node dist/stdio.js`, also the `bin` entry). This is the default
  for Claude Desktop and Claude Code.
- **Streamable HTTP** (`node dist/http.js`, port from `PORT`, default 3000). It
  is stateless: each POST to `/mcp` gets a fresh server and transport.
  `GET /health` returns `{ "status": "ok" }`. It uses Node's `http` module, so
  there is no Express dependency.

## Docker

The Dockerfile builds in two stages: a `node:22-alpine` stage builds, and a
smaller runtime stage copies `dist/` and the production dependencies. It runs
as the non-root `node` user, `EXPOSE 3000`, and the default command serves HTTP.
Docker is not installed on the dev machine, so CI is where the image build is
proven.

## Testing

- **Unit:** `resolveTeam` (abbreviation, nickname, city, id, ambiguous, none);
  cache TTL behaviour; client error paths (non-200, timeout, bad JSON).
- **Tools:** each tool against a fake `EspnClient` serving fixtures, with
  success, ambiguous-team and upstream-error cases.
- **Integration:** `createServer()` plus the SDK's `InMemoryTransport` and a
  real `Client`. It lists tools, resources and prompts, and calls each tool
  end to end.
- Fixtures are real ESPN responses, recorded once and trimmed to what the code
  reads. Tests never touch the network.

## CI

GitHub Actions on push and PR runs `npm ci`, lint, `tsc --noEmit`, `vitest run`
and `docker build`.

## README

- What it is and why
- Quick start
- Config snippets for Claude Desktop and Claude Code
- An example conversation
- The tool, resource and prompt reference
- Architecture notes (layering, why the tools take a team name, stateless HTTP)
- The unofficial-API note
- Licence: MIT

## Out of scope

Player-level stats, live play-by-play, betting odds, authentication on the HTTP
endpoint, persistent storage, other sports.
