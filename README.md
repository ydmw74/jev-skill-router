# jev-skill-router

Skill suggestion for Hermes via the TypeSafe cookbook pattern (Jev) — as a
stdio MCP server. Suggests **at most one** skill for a request, before the
agent loads anything.

```
user request ──▶ skill_select ──▶ Jev (TypeSafe Decisions API)
                                  1. rank all skills + "need a skill?" gates
                                  2. rerank top-3 with full descriptions
                  ◀── one skill name + <skill_relevance> block
```

## Why

Hermes truncates skill index descriptions to ~60 characters. At 190+ skills
the agent guesses from near-zero information — the `.pptx` *editing* skill
reads like the *authoring* one, and on turns where nothing fits, the roster's
"err on the side of loading" invites a wrong load anyway. Measured on the
Hermes catalog (TypeSafe's cookbook, 488 requests, jev-1.12): wrong skill
loads **16.8% → 7.3%**, needless loads **9.8% → 4.0%**. Each avoided load
saves 5–15k characters of context ballast; each wrong one wastes a turn.

## How (two Jev calls per request)

1. **Rank + gate** — one `Choice` question over ALL skill names (criteria =
   the index description) plus 3 `Noul` gates asking whether the turn wants
   an action at all (`acts_on_user_system`, `would_follow_documented_procedure`,
   inverted `prose_suffices`). Gate mean < 0.30 → suggest nothing.
2. **Rerank** — the top-3 candidates return with their FULL description +
   700 chars of `SKILL.md`, one `fits`-Noul each (all < 0.30 → abstain).

The winner's name goes into a `<skill_relevance>` block the caller can put
AFTER the skill roster — the roster itself stays byte-stable, so prefix
caching holds. An `abstain` is a valid answer; the agent keeps its own
judgement ("Ignore this if it does not fit what the user actually asked for").

The roster loads live from `$HERMES_HOME/skills` per session — never
hardcoded. Symlinked skills are followed, block-scalar (`>` / `|`)
descriptions are parsed, category/top-level duplicates are deduped.

## Tool

```
mcp__jev_skill_router__skill_select(request: str)
→ {decision: suggest|abstain, suggested: str|null, gate, gate_values,
   shortlist, fits, suggestion_block, usage_call1, usage_call2, seconds}
```

## Setup

Requires Python 3.10+ and [uv](https://docs.astral.sh/uv/) (or pip).

### Direct mode (standalone — your own TypeSafe key)

```yaml
  jev-skill-router:
    command: <uv path>
    args: [run, --project, <path/to/jev-skill-router>, jev-skill-router, mcp]
    env:
      JEV_TYPESAFE_API_KEY: <your key from console.typesafe.ai>
    timeout: 60
    enabled: true
```

Get a key at `console.typesafe.ai/keys` ($5 free credits at signup).
Cost: ≈ $0.00004 per two-call selection, 0.3–0.9 s latency.

### Adapter mode (central endpoint, shared key)

If you run a central Jev endpoint (recommended when several hosts share one
key), point the server at it instead — the payload format is identical:

```yaml
    env:
      JEV_SKILL_ADAPTER_URL: http://<your-adapter>:3004
      JEV_SKILL_ADAPTER_TOKEN: <adapter bearer token>
```

Adapter mode wins when `JEV_SKILL_ADAPTER_URL` is set. Works with any
endpoint that forwards `{state, questions}` to the TypeSafe Decisions API
(`api.typesafe.ai/v1/systemone`) and returns the raw answer.

## Verify

```
hermes mcp test jev-skill-router
```

Example results (German paraphrase, zero lexical overlap with the skill
name): „Schau mal in die Kasse und prüfe den Kontostand" → `n-ahv-kasse`
(fits 0.96); „Erklär mir was ein Monad ist" → abstain (gate 0.07).

## Notes

- Model: `jev-latest` via the Decisions API (`system_one`). Two calls are
  the cookbook shape; the first call's ranking carries most of the value.
- Thresholds (0.30/0.30) and SHORTLIST=3 follow the cookbook defaults.
- Works for any MCP host, not just Hermes: the roster path and the tool
  schema are the only Hermes-specific parts.

## License

MIT