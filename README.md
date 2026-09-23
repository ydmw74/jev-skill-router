# jev-skill-router

Skill suggestion for Hermes via the TypeSafe cookbook pattern (Jev) — as a
stdio MCP server. Suggests **at most one** skill for a request, before the
agent loads anything.

## Why

Hermes truncates skill index descriptions to ~60 characters. At 190+ skills
the agent guesses from near-zero information — the `.pptx` *editing* skill
reads like the *authoring* one, and on turns where nothing fits, the roster's
"err on the side of loading" invites a wrong load anyway. Measured on the
Hermes catalog (TypeSafe cookbook, 488 requests): wrong skill loads
**16.8% → 7.3%**, needless loads **9.8% → 4.0%** — each avoided load saves
5–15k characters of context ballast.

## How (two Jev calls per request)

1. **Rank + gate** — one Choice question over ALL skill names (criteria =
   index description) plus 3 Noul gates asking whether the turn wants an
   action at all (`acts_on_user_system`, `would_follow_documented_procedure`,
   inverted `prose_suffices`). Gate mean < 0.30 → suggest nothing.
2. **Rerank** — the top-3 candidates return with their FULL description +
   700 chars of `SKILL.md`, one `fits`-Noul each (all < 0.30 → abstain).

The winner's name goes into a `<skill_relevance>` block the caller can put
AFTER the skill roster — the roster itself stays byte-stable, so prefix
caching holds.

The roster loads live from `HERMES_HOME/skills` per session — never
hardcoded. Symlinked skills are followed, block-scalar (`>` / `|`)
descriptions are parsed, category/top-level duplicates are deduped.

## Tool

```
mcp__jev_skill_router__skill_select(request: str)
→ {decision: suggest|abstain, suggested: str|null, gate, gate_values,
   shortlist, fits, suggestion_block, usage_call1, usage_call2, seconds}
```

## Wiring (Mac; profile configs likewise)

```yaml
  jev-skill-router:
    command: /Users/markus/.local/bin/uv
    args: [run, --project, /Users/markus/Hermes/Homelab/jev-skill-router, jev-skill-router, mcp]
    env:
      JEV_SKILL_ADAPTER_URL: http://utap.amhomenet.de:3004
      JEV_SKILL_ADAPTER_TOKEN: <LAN token — NOT the TypeSafe key>
```

Backend: the `jev-adapter` on utap:3004 (passthrough mode, `ydmw74/jev-adapter`)
forwards `{state, questions}` 1:1 to the TypeSafe Decisions API. Provider key
stays server-side; this server holds only the LAN adapter token. Cost:
≈ $0.00004 per two-call selection, ~0.3–0.9 s.

## Verified live (2026-09-23)

| Request | Result |
|---|---|
| „Schau mal in die Kasse und prüfe den Kontostand der AHV" | `n-ahv-kasse`, fits 0.96 (fints-banking 0.84 runner-up) |
| „Erstelle mir eine Präsentation als pptx" | `powerpoint`, fits 0.95 |
| „Erklär mir was ein Monad ist" | gate abstain (0.07) — no suggestion |

E2E through a fresh `hermes chat -q` session: the agent called
`skill_select` and loaded the suggested skill.

## Deployment to opcl1

`git clone` to `/home/hermes/jev-skill-router`, `uv sync`, same YAML block
(paths adjusted, `HERMES_HOME=/home/hermes/.hermes`), `hermes mcp test`,
gateway restart. See `docs/mcp-servers.md` in `ydmw74/myHermesAgent`.