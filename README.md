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

### 1. Clone and sync

```bash
git clone https://github.com/ydmw74/jev-skill-router.git
cd jev-skill-router
uv sync          # creates .venv; alternatively: pip install -e .
```

Requires Python 3.10+ and [uv](https://docs.astral.sh/uv/).

### 2. Get a TypeSafe API key

Create one at `console.typesafe.ai/keys`.

### 3. Register in Hermes

Add to `~/.hermes/config.yaml` (or a profile config) under `mcp_servers`:

```yaml
  jev-skill-router:
    command: <uv path>          # e.g. /Users/you/.local/bin/uv
    args: [run, --project, <path/to/jev-skill-router>, jev-skill-router, mcp]
    env:
      JEV_TYPESAFE_API_KEY: <your key>
    timeout: 60
    enabled: true
```

Hermes discovers the tool on the next session start. Alternatively ask your
Hermes agent to add the block for you ("add this MCP server to my config")
— it edits the same YAML.

> **Note on the Agent Plugins v1 package (`hermes plugins install`):** the
> portable loader does not interpolate `${JEV_TYPESAFE_API_KEY}` in
> `mcp.json` — it only expands `${PLUGIN_ROOT}`/`${PLUGIN_DATA}`. For the
> packaged install, `JEV_TYPESAFE_API_KEY` must already be set in the
> environment the MCP server inherits (Hermes tracks this gap in
> NousResearch/hermes-agent#120526). The native `mcp_servers` block above
> does not have this limitation — declare the env there if in doubt.

### 4. Install the companion skill (recommended)

The skill tells the agent WHEN to call `skill_select` (and that an `abstain`
is a valid answer). Copy it into your Hermes skills directory:

```bash
mkdir -p ~/.hermes/skills/devops/jev-skill-router-mcp
cp skill/SKILL.md ~/.hermes/skills/devops/jev-skill-router-mcp/SKILL.md
```

### 5. Verify

```bash
hermes mcp test jev-skill-router
```

Cost: see current TypeSafe pricing; each selection is two small Decisions
calls. Latency: 0.3–0.9 s per selection.

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

## Example results

Paraphrases with zero lexical overlap with the skill name:

- „Bearbeite die Folien meiner Präsentation von letzter Woche" → `powerpoint`
  (fits 0.90) — a semantic match the lexical index cannot see
- „Was ist eine Cloud-Funktion im Vergleich zu einem Server?" → abstain
  (gate 0.05) — a prose question needs no skill

## Notes

- Model: `jev-latest` via the Decisions API (`system_one`). Two calls are
  the cookbook shape; the first call's ranking carries most of the value.
- Thresholds (0.30/0.30) and SHORTLIST=3 follow the cookbook defaults.
- Works for any MCP host, not just Hermes: the roster path and the tool
  schema are the only Hermes-specific parts.

## License

MIT