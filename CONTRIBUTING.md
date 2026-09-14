# Contributing skills to the JARVIS OS community catalog

This repository is the **community catalog** that JARVIS OS reads when you open
**Settings → Agents → Community skills → Browse catalog**. Anyone can publish a
skill here: it's one JSON file plus one entry in `catalog.json`, shipped with a
pull request. No build step, no code, no registration.

A **skill** teaches JARVIS a repeatable job — *"do a security sweep"*, *"update
dependencies and run the tests"* — as a short sequence of steps executed by the
agent with its real tools (file search, shell, browser…). If you can describe
the job in 1–8 sentences, you can write a skill.

---

## Repository layout

```
catalog.json          ← the index JARVIS fetches (this is the catalog)
skills/<id>.json      ← one file per skill, raw JSON
```

When JARVIS opens the catalog it downloads `catalog.json`, lists the entries in
the Community skills UI, and downloads a skill's file only when the user clicks
**Install**.

---

## The skill file format

A skill is a single JSON file. Here is a complete, valid example
([skills/security-sweep.json](skills/security-sweep.json)):

```json
{
  "name": "security-sweep",
  "description": "Scan the project for common vulnerabilities (secrets, injection, unsafe deps) and report findings with file:line references.",
  "version": "1.0.0",
  "author": "your-github-name",
  "trigger": ["security sweep", "security check"],
  "inputs": [],
  "steps": [
    { "title": "Search the codebase for hardcoded secrets, API keys and tokens", "role": "debugger" },
    { "title": "Look for injection risks: unsanitized input reaching queries, commands or HTML", "role": "debugger" },
    { "title": "Check dependency manifests for known-risky patterns and note versions", "role": "tester" },
    { "title": "Write the findings report: severity, file:line, and a one-line fix for each", "role": "reviewer" }
  ],
  "allowed_tools": ["fs_search", "fs_read_file", "code_search_symbols", "fs_list_dir", "term_run"],
  "validation": "every finding has a real file:line reference verified by reading the file",
  "expected_output": "findings table with severity + location + suggested fix",
  "failure_recovery": "report which checks could not run and why; never invent findings",
  "test_cases": []
}
```

### Field reference

| Field | Required | Type | What it does |
|---|---|---|---|
| `name` | **yes** | string | Identifier and display name. Non-empty. Install derives the local filename from it (`my-skill.json`), so prefer lowercase-kebab-case. |
| `description` | **yes** | string | One sentence: what the skill does and what it produces. Shown in the catalog UI. Non-empty. |
| `version` | **yes** | string | Semver-ish (`"1.0.0"`). Bump it when you change a published skill. |
| `steps` | **yes** | array | **1 to 8 steps** — the validator rejects more. Each step is `{ "title", "role" }`. |
| `author` | no | string | Your GitHub handle. |
| `trigger` | no | string[] | Phrases that auto-match this skill when the user's request contains one (case-insensitive), e.g. `"build and test"`. |
| `inputs` | no | array | Ask-the-user parameters: `{ "name": "target", "required": false }`. |
| `allowed_tools` | no | string[] | Tool names the skill expects to use. Metadata/hints — role allow-lists still gate actual use. |
| `validation` | no | string | How the agent can check its own work succeeded. |
| `expected_output` | no | string | Shape of the final deliverable (report, table, file…). |
| `failure_recovery` | no | string | What to do when a step fails, so the agent degrades honestly instead of guessing. |
| `test_cases` | no | array | Reserved for future automated skill testing. |

### Step titles

Write each `title` as an imperative instruction the agent can act on directly —
it becomes that step's task verbatim:

- ✅ `"Search the codebase for hardcoded secrets"` (a verb, a scope, a goal)
- ❌ `"Security stuff"` (too vague — the agent will improvise unpredictably)

### Roles — pick the right one per step

Each step runs under a role that shapes the agent's system prompt. Valid
values (anything else is rejected):

| Role | The agent's behavior in that step |
|---|---|
| `coder` | Implements with minimal, focused edits. |
| `debugger` | Gathers **evidence first** (searches code, reads logs), then concludes. Use for any "find / analyze / scan" work. |
| `reviewer` | **Strictly read-only.** Inspects diffs and files, reports findings. Use for any final "report / summarize / review" step. |
| `tester` | Discovers the project's build/test commands and runs them. Use for "verify / run the tests" steps. |
| `research` | Answers from available material and its own knowledge, citing sources. |

A healthy pattern for multi-step skills: evidence (`debugger`) → action
(`coder`) → verification (`tester`) → report (`reviewer`).

---

## The catalog entry

Add one object to the `items` array in `catalog.json`, in alphabetical order by
`id`:

```json
{
  "kind": "skill",
  "id": "security-sweep",
  "name": "Security Sweep",
  "description": "Scan the project for common vulnerabilities (secrets, injection, unsafe deps) and report findings with file:line references.",
  "author": "jarvis-os",
  "url": "https://raw.githubusercontent.com/chi5sb/jarvis-os-catalog/main/skills/security-sweep.json"
}
```

- `kind` must be `"skill"` (other kinds may come later).
- `url` must point at the **raw** file on your published branch. A normal
  `github.com/.../blob/...` URL also works — JARVIS converts it automatically —
  but raw is preferred.
- `description` in the entry should match the skill's own description.

---

## How installation works (good to know)

1. User clicks **Install** → JARVIS downloads the `url`.
2. The file is parsed and **validated**: non-empty `name`/`description`,
   **1–8 steps**, every step's `role` one of the five roles above. A file that
   fails validation is rejected with the reason.
3. The skill is saved locally as `<sanitized-name>.json` and is **active
   immediately** — no restart. Re-installing overwrites, which is also how
   users update.

Users can also paste **any** GitHub URL into the Community skills box and
install it directly — your skill doesn't have to be in this catalog to work,
but catalog inclusion is what makes it discoverable.

---

## Submitting a skill

1. Fork this repo.
2. Add `skills/<your-id>.json` following the format above.
3. Add its entry to `catalog.json`.
4. Validate your JSON: `python -m json.tool skills/<your-id>.json` (or any JSON
   linter) — both files must parse.
5. Open a pull request. A maintainer reviews for safety and quality and merges.

### PR checklist

- [ ] Skill file and `catalog.json` both parse as JSON
- [ ] 1–8 steps, each with a valid role (`coder`/`debugger`/`reviewer`/`tester`/`research`)
- [ ] Step titles are imperative and specific
- [ ] No skill that exfiltrates data, damages systems, or requires disabling JARVIS's sandbox/approvals
- [ ] `version` bumped if you're changing an existing skill
- [ ] Catalog entry added, sorted by `id`, `url` points at the raw file

### Quality tips

- **Fewer, sharper steps beat many vague ones.** Each step is one agent turn.
- Say what *done* looks like (`validation`, `expected_output`) — it measurably
  improves results.
- Tell the agent what to do when things fail (`failure_recovery`); "never
  invent findings" is a good default for analysis skills.
- Test your skill for real: paste its raw URL into JARVIS
  (Settings → Agents → Community skills) and run it against a scratch project
  before opening the PR.
