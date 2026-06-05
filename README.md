# hypernet-audit-anchors

**AnchoredChain audit anchor sink** for the Hypernet T.4 token-accounting audit ledger.

This **public, append-only, write-protected** repository is the external anchor sink that makes the
`AnchoredChain` recompute-detection real (see `2.7.23.1` spec in the main Hypernet archive). It exists
because a tamper-evident audit chain on a single machine is recompute-forgeable by a dishonest local
writer; the defense requires a checkpoint of the chain's `(head, count)` stored **outside that
writer's authority**. This repo is that "outside." (It is public so GitHub branch protection actually
enforces no-force-push / no-delete — the free tier does not enforce protection on private repos, and
public also serves the Hypernet transparency thesis: the audit trail is openly verifiable.)

It follows the `2.7.22` AI-Owned-Repository pattern and is the **Wave 4 (`2.7.30`) anchor-sink
prototype** — the same writer protocol the future Hypernet Server's `audit_anchors` table will use,
with a different backend.

## What lives here

```
audit-log/
  anchor-000001.json   # one AnchorRecord per file, each added by its own immutable commit
  anchor-000002.json
  ...
```

The medium is **commit history, not the latest tree**. Each anchor is written as **its own file** in
**its own add-commit** and is **never rewritten**. Each file is one `AnchorRecord`:

```json
{"head":"<chain_state of the last anchored row>","count":<#rows anchored>,
 "prev_head":"<prior anchor's head>","prev_count":<prior anchor's count>,
 "algorithm":"anchored-unkeyed-sha256","ts":<unix epoch>}
```

The records are **anchor-chained** (each commits to the prior anchor's `(head, count)`) **and** each
lives in its own immutable add-commit. Verification reads each anchor **from the commit that added
it** (`git log --diff-filter=A` + `git show <commit>:<file>`), not from whatever blob sits at the
branch tip. Combined with branch protection (linear history, no force-push, no deletions), this
detects the *recompute-then-extend* attack: a later normal fast-forward commit can rewrite or delete
the latest-tree files, but it **cannot alter the earlier add-commit blobs** that verification reads —
so the original anchor still pins the original chain prefix.

> **Why not one append-only `anchors.jsonl`?** That was the round-1 design and it was **broken**:
> branch protection blocks force-push but *not* a normal fast-forward commit that rewrites that one
> file, so a write-scoped PAT holder could overwrite it with a forged log matching a recomputed
> ledger — no force-push needed. Commit-history-as-medium closes that hole (Codex cross-vendor
> red-team, rounds 1–2; see `2.7.23.2`).

## Trust model

- **Append-only is the load-bearing property.** It is enforced by GitHub branch protection
  (`require_linear_history`, `allow_force_pushes:false`, `allow_deletions:false`), not by convention.
- **Outside the metered writer's authority.** The local Hypernet Agent writes anchors here via a
  scoped fine-grained PAT (write to this repo only). It cannot rewrite history (force-push blocked),
  so it cannot launder a recompute.
- **Founder break-glass.** `enforce_admins:false` leaves the repo owner (Matt) an admin override for
  recovery — an audited, reverse-transparent exception (`2.7.22`).
- **No secrets in this repo.** The PAT lives only in the Agent's gitignored `secrets/config.json`.

## How to verify

Clone this repo (the authoritative immutable history), then run `AnchoredChain.verify()` against the
local token-accounting ledger with this sink: the sink reconstructs the anchor log by reading each
`audit-log/anchor-*.json` **from the commit that added it**; every anchored prefix must still match
the live chain, and the anchor log must be internally chained. A recompute or truncation of the
ledger that diverges from any anchored `(head, count)` is detected.

See `Hypernet Structure/2 - AI Accounts/2.7 - AI Shared Understanding/2.7.23.1` (and `2.7.23.2`, the
GitHubSink spec) in the main Hypernet archive for the full design and threat model.

— Created 2026-06-05 by Tally (`2.4.1`), Master Librarian, per Matt's directive.
