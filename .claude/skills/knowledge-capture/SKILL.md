---
name: knowledge-capture
description: >
  Records a decision you reached while reading code as a candidate micro-decision, via the
  `kcapture` transport (a CLI + stdio MCP) that forwards a capture manifest to `kmint` — the
  single deterministic writer. It writes nothing itself and never publishes: units are born
  `SubmittedForRefinement` and the tool-approval prompt is the human gate (ADR-KCM-FUNC-0003,
  the `reverse-engineering.code` binding). Use this skill whenever the user wants to capture,
  record, remember, or "write down" a decision, rule, or pattern discovered in a codebase, or
  to register the capture tool in this host. Trigger on: "capture this decision", "record this
  as knowledge", "remember this rule", "log this micro-decision", "save this to the ledger",
  "add a knowledge capture", "install the capture tool", "set up kcapture". It is a thin
  reference to the published `kcapture` package (papeete-consulting/kcapture) — no local logic.
---

# Knowledge Capture

You turn a decision the user reached **about code** into a *candidate* micro-decision, by
building a **capture manifest** and forwarding it to **`kmint`** through the **`kcapture`**
transport. You never write the ledger yourself, and nothing you capture is ever published:
every unit is born `SubmittedForRefinement`, and the host's tool-approval prompt on
`record_capture` **is the human gate**. This is the `reverse-engineering.code` source-type
binding — born LLM-sourced, human-gated (ADR-KCM-FUNC-0003).

> **You do not write, and you do not publish.** `kmint` is the single deterministic writer;
> `kledger` is read-only. Your job is to propose an honest manifest and let the human approve
> the forward. If the capture is wrong, fix the manifest — never the ledger.

---

## Step 0 — Prerequisites

This skill is a thin reference to the **`kcapture`** package
(`papeete-consulting/kcapture`). Ensure it is installed and registered in this host (idempotent):

```bash
command -v kcapture || pip install "kcapture==0.0.1"
kcapture install --host claude          # writes ./.mcp.json (mcpServers → kcapture serve-mcp via uvx)
```

`kcapture install` is idempotent — safe to re-run. It launches the MCP over stdio via
`uvx --from <pin> kcapture serve-mcp` (zero pre-install). After registration this host exposes
three tools: `list_source_types`, `validate_manifest`, `record_capture`. You can also drive the
CLI directly: `kcapture validate|record|list-source-types`.

---

## Your workflow

1. **Pick the source type.** Call `list_source_types` (or `kcapture list-source-types`). For a
   decision recovered from source code it is almost always **`reverse-engineering.code`**.
2. **Build the manifest.** The contract is owned by `kmint` (kcapture validates against the
   schema it ships). Fill:
   - `sourceType` — the label from step 1.
   - `domain` — the MD domain segment (upper-alphanumerics). `kmint` mints the full canonical
     handle `<PREFIX>.KNOW.MD.<DOMAIN>.<NNN>`, where `<PREFIX>` is the enterprise/scope context
     of the consuming repo — you supply only `<DOMAIN>`. Ask the user if unsure.
   - `source` — the **immutable code pin**: `{ "kind": "code", "repoUrl", "commit": "<full sha>", "path" }`.
   - `derivedFrom` — the **two-capture case**: when your own analysis produced the units,
     capture that LLM output as `{ "kind": "llm-output", "text": "…what you inferred…" }`. It is
     recorded as a separate capture `DERIVED_FROM` the code (FUNC-0003 R3) — the provenance the
     human gate reviews. Do **not** fold it into the unit text.
   - `units` — one entry per decision: `{ "text": "the recovered decision, in one sentence" }`.
     Do not assign provider/confidence — `kmint` derives them.
   - `trigger` — `{ "mode": "model-proposed", "by": "claude:knowledge-capture@<model>", "at": "<ISO-8601>" }`.
     **Keep `trigger.mode` honest**: `model-proposed` when you proposed it (URBA-0003 P5).
3. **Validate first.** `validate_manifest` (or `kcapture validate -`). Fix every error before
   proposing the write — an invalid manifest is rejected pre-write by `kmint`.
4. **Propose the write, let the human approve.** Call `record_capture` (or
   `kcapture record -`). **Ask before** doing so — the approval prompt is the gate. Optionally
   run with `dry_run` first to show the exact `kmint ingest` command without writing.
5. **Report the result honestly.** On success `kmint` returns the minted
   `SubmittedForRefinement` iris. Say the units are **candidates awaiting human refinement** —
   never claim anything was published.

---

## Guardrails

- **Never claim a capture was published.** Units are born `SubmittedForRefinement`; publication
  is a separate human-authority act outside this tool (FUNC-0003 R1/C2).
- **The approval prompt is the gate.** Always let the human approve `record_capture`; never
  auto-forward. `trigger.mode` must reflect reality.
- **Two captures, not one.** Code and "what the model said about it" are distinct captures
  linked `DERIVED_FROM`; keep them separate (R3).
- **You are a transport.** No write logic lives here — everything runs through the pinned
  `kcapture` → `kmint`. If the schema rejects the manifest, fix the manifest.
- **kledger is read-only.** To read a captured candidate back, resolve it — never write through
  the resolver.
