---
name: brcs
description: Triage and merge open PRs on the bitcoin-sv/BRCs repository. Keeps the BRC number each PR assigned itself unless that number is already taken; only then reassigns (with the user's approval) to the lowest available number. Renames files when needed, wires links into README.md / SUMMARY.md / dir README. Use when user says "/brcs", "merge BRC PRs", "triage BRCs", "process BRC pull requests", or wants to drain the open PR queue on the BRCs docs repo.
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - AskUserQuestion
  - TaskCreate
  - TaskUpdate
  - TaskList
---

# /brcs — Triage and Merge BRC PRs

Drain open PRs on bitcoin-sv/BRCs. Assign next BRC numbers. Wire indices. Push.

## Preconditions

- `cwd` is a clone of `bitcoin-sv/BRCs` (check `git remote -v`).
- `gh` authed, push access to `origin/master`.
- On `master`, clean (or only `.DS_Store` untracked).

If preconditions fail → tell user, stop.

## Pipeline

```
list PRs → read README index for highest BRC# → triage each PR →
  (close duplicates | merge content updates | merge new BRCs with renumber if needed) →
  resolve index conflicts → push → report
```

## Step 1 — Snapshot

```bash
gh pr list --state open --limit 50
```

Read `README.md` "## Standards" table. Highest existing `NNN | [Title](path)` row = `LAST_BRC`.

**The PR's own number is authoritative unless it collides.** Authors pick numbers deliberately — often memetic or self-referential (e.g. `0069`, `0100`, matching a sibling BRC). Never renumber just because a lower gap exists or because the number is above `LAST_BRC`. A number is only reassigned when it is *taken*: a file with that number already exists on disk, or an earlier PR in this same run has claimed it.

The `TAKEN` set and the free list both come from the files actually on disk:

```bash
find . -path './.git' -prune -o -name "[0-9]*.md" -print \
  | grep -oE "[0-9]{4}\.md" | grep -oE "^[0-9]{4}" | sed 's/^0*//' | sort -n -u > /tmp/nums.txt
LAST_BRC=$(tail -1 /tmp/nums.txt)
comm -23 <(seq 1 "$LAST_BRC" | sort) <(sort /tmp/nums.txt) | sort -n > /tmp/free_brc.txt
echo "LAST_BRC=$LAST_BRC  next free: $(head -1 /tmp/free_brc.txt)"
```

`TAKEN` = `/tmp/nums.txt` (every number on disk). `FREE_LIST` = `/tmp/free_brc.txt` (ascending gaps in `1..LAST_BRC`).

`FREE_LIST` is **only** consulted for collisions. It is the fallback allocator, not the default one. When it is empty, fall back to `LAST_BRC + 1`. Add each merged BRC's number to `TAKEN` as you go, so a later PR in the run collides against it correctly.

Also note structure:
- Per-category dir READMEs: `apps/README.md`, `wallet/README.md`, `transactions/README.md`, `payments/README.md`, `overlays/README.md`, `peer-to-peer/README.md`, `key-derivation/README.md`, `outpoints/README.md`, `opinions/README.md`, `tokens/README.md`, `scripts/README.md`, `state-machines/README.md`.
- `SUMMARY.md` (GitBook index) groups by category heading.
- Top-level `README.md` has flat `BRC | Standard` table.

## Step 2 — Triage each PR

For each open PR (ascending creation date), inspect:

```bash
gh pr view <N> --json number,title,state,mergeable,files,headRefName,baseRefName,isDraft
gh pr diff <N> --patch | head -60
git fetch origin pull/<N>/head:pr-<N>
```

Classify into one of:

### (a) DRAFT
Skip. Mention in final report.

### (b) Duplicate — content already in master
Symptoms:
- Adds `XXX.md` where same-named file already exists with byte-identical content (modulo BRC number references), OR
- Modifies existing file but `diff <(git show pr-<N>:path) path` is trivial (whitespace/formatting only).

Action: confirm with user; on approval, `gh pr close <N> --comment "Closing — content already merged to master as BRC-YYY (path/, commit ZZZZZ). ..."`.

### (c) Content update to existing BRC
PR's latest commit modifies an existing BRC file with non-trivial changes (rework, clarifications, field renames). No new BRC number needed.

Action: `git merge --no-ff --no-commit pr-<N>`. Resolve conflicts (see Step 4). Also update any title-bearing references in `SUMMARY.md` + dir README if the BRC's title changed. Commit with body referencing PR.

### (d) New BRC
PR adds a new `<dir>/NNNN.md` claiming number `PROPOSED`.

**Default: keep `PROPOSED`.** Check only for collision:

```bash
grep -qx "$PROPOSED" /tmp/nums.txt && echo COLLISION || echo FREE
```

(also treat it as a collision if an earlier PR in this run was assigned `PROPOSED`).

- **FREE** → `ASSIGNED = PROPOSED`. Merge as-is. No rename, no `sed`, no session mapping entry. This holds even when `PROPOSED` is far above `LAST_BRC` or leaves gaps below it — leaving gaps is fine and expected.
- **COLLISION** → the number is genuinely unusable. Before touching anything, **ask the user** via `AskUserQuestion`:
  - state which existing file/PR holds `PROPOSED`,
  - offer the lowest available number (head of `FREE_LIST`, else `LAST_BRC + 1`) as the recommended option,
  - note that authors often pick numbers for a reason, so the user may prefer another number or to ask the author.

  Renumbering is never automatic — a collision is a question, not a decision. On approval:
  1. `ASSIGNED` = the number the user chose; pop it from `FREE_LIST` if it came from there.
  2. `git mv <dir>/<PROPOSED>.md <dir>/<ASSIGNED>.md` in the PR's branch (or apply rename after merge). Filenames are zero-padded to 4 digits (`0141.md`).
  3. In the renamed doc, `sed -i '' "s/BRC-<PROPOSED>/BRC-<ASSIGNED>/g"` (or Edit) — but **only the document's self-references** (title heading + any "this BRC" / "BRC-XXX" pointers pointing at itself). Do NOT change references to *other* BRCs.
  4. Track the mapping `PROPOSED → ASSIGNED` for this session.
  5. Before merging each subsequent PR, grep its files for any reference to a `PROPOSED` number that has since been remapped and rewrite to `ASSIGNED`.

Action: merge, resolve conflicts, update indices (Step 5), commit. Add `ASSIGNED` to `TAKEN`; bump `LAST_BRC` if `ASSIGNED > LAST_BRC`.

**Note on gaps:** gaps in `1..LAST_BRC` are left alone. A gap may exist because that number was withdrawn or is informally reserved, so it is only ever filled as the fallback target of an approved collision fix — never proactively, and never to "tidy up" the numbering.

## Step 3 — Track BRC number mapping

Maintain a session map (mental or scratch file) of `PROPOSED → ASSIGNED` for every PR processed in this run. Other PRs in the queue may cross-reference these numbers; rewrite references when you encounter them.

Check cross-references:

```bash
git show pr-<N>:<file> | grep -oE "BRC-[0-9]+" | sort -u
```

For each match, if the number appears in the session map under `PROPOSED`, plan to rewrite the reference to `ASSIGNED` before commit.

## Step 4 — Resolve index conflicts

PRs typically modify the same lines in `README.md`, `SUMMARY.md`, and the dir's `README.md`. After `git merge --no-ff --no-commit`, conflicts appear in those files.

For the common case (each PR appends a single new line, conflict between HEAD's growing list and PR's single line):

```bash
sed -i '' '/^<<<<<<< HEAD$/d; /^=======$/d; /^>>>>>>> pr-/d' README.md SUMMARY.md <dir>/README.md
```

This strips the markers, keeping HEAD's lines followed by PR's new line — correct order.

Verify:

```bash
grep -n "<<<<<<<\|>>>>>>>\|=======" README.md SUMMARY.md <dir>/README.md || echo clean
```

If any PR pulled in unrelated stale changes (e.g. removed sections because its branch base predates a refactor on master), `git checkout HEAD -- <file>` and re-add the intended one-line addition manually.

## Step 5 — Wire indices

Every new BRC needs entries in **three** places (four if there's a dir README beyond the standard set).

**All three indices are sorted ascending by BRC number.** Since the PR keeps its own number, the entry often does sort last — but not always (a number below `LAST_BRC` lands mid-table). Check where it belongs and insert in sorted position rather than assuming append. Use `Edit` anchored on the surrounding rows (`sed` with `|` delimiters breaks on the table's own pipes).

1. **`README.md`** top-level table — insert row into the `BRC | Standard` table in numeric order:
   ```
   NNN  | [Title](./<dir>/NNNN.md)
   ```

2. **`<dir>/README.md`** category table — insert into its `BRC | Standard` table in numeric order:
   ```
   NNN  | [Title](./NNNN.md)
   ```

3. **`SUMMARY.md`** — insert under the matching `## <Category>` section in numeric order:
   ```
   * [Title](./<dir>/NNNN.md)
   ```

PR authors usually update some but not all. Diff PR's `files` list against this 3-place requirement, move any misplaced row into sorted position, and add missing entries manually before commit.

Caveat for Step 4: the strip-conflict-markers `sed` keeps `HEAD`'s lines then the PR's line — correct only when the new row genuinely sorts last. When the number sorts mid-table, strip the markers then move the row into place, or drop the PR's row and re-add it manually in sorted position.

## Step 6 — Commit

```bash
git commit -m "$(cat <<'EOF'
feat: add BRC-NNN <Title> (#<PR>)

Merges PR #<PR> from <head_ref>.

Co-authored-by: <Author Name> <author@example.com>
EOF
)"
```

For content updates use `refactor:` or `docs:` instead. Always reference the PR number.

## Step 6.5 — Verify index completeness (MANDATORY before push)

Run after all PRs are committed. Catches any BRC file not wired into all three indices.

```bash
# All numbered BRC files on disk
find . -path './.git' -prune -o -name "[0-9]*.md" -print | sed 's|^\./||' | sort > /tmp/all_brc_files.txt

# README.md links
grep -oE "\./[a-z-]+/[0-9]+\.md" README.md | sed 's|^\./||' | sort > /tmp/readme_links.txt

# SUMMARY.md links
grep -oE "\./[a-z-]+/[0-9]+\.md" SUMMARY.md | sed 's|^\./||' | sort > /tmp/summary_links.txt

# Files on disk but missing from top-level README.md
echo "=== Missing from README.md:"
comm -23 /tmp/all_brc_files.txt /tmp/readme_links.txt

# Files on disk but missing from SUMMARY.md
echo "=== Missing from SUMMARY.md:"
comm -23 /tmp/all_brc_files.txt /tmp/summary_links.txt

# Per-dir README check
for dir in apps wallet transactions payments overlays peer-to-peer key-derivation outpoints opinions tokens scripts state-machines; do
  files=$(find ./$dir -name "[0-9]*.md" 2>/dev/null | grep -oE "[0-9]+\.md" | sort)
  linked=$(grep -oE "\./[0-9]+\.md" ./$dir/README.md 2>/dev/null | grep -oE "[0-9]+\.md" | sort)
  missing=$(comm -23 <(echo "$files") <(echo "$linked"))
  [ -n "$missing" ] && echo "=== $dir/README.md missing: $missing"
done
```

Also verify each index is still sorted ascending (gap-filling inserts mid-table):

```bash
# Top-level README.md: one global ascending table
nums=$(grep -oE "\./[a-z-]+/[0-9]+\.md" README.md | grep -oE "[0-9]+" | sed 's/^0*//')
diff <(echo "$nums") <(echo "$nums" | sort -n) >/dev/null || echo "=== README.md rows out of order"

# SUMMARY.md: ascending WITHIN each "## Category" section only
grep -nE "^## |\./[a-z-]+/[0-9]+\.md" SUMMARY.md \
 | sed -E 's/^[0-9]+:## (.*)$/SEC\t\1/; s/^[0-9]+:.*\/0*([0-9]+)\.md.*$/NUM\t\1/' \
 | awk -F'\t' '$1=="SEC"{sec=$2;last=0;next} $1=="NUM"{n=$2+0; if(n<last) print "=== SUMMARY.md out of order in ["sec"]: "n" after "last; last=n}'

# Per-dir READMEs: ascending within each file
for dir in apps wallet transactions payments overlays peer-to-peer key-derivation outpoints opinions tokens scripts state-machines; do
  nums=$(grep -oE "^[0-9]+ " ./$dir/README.md 2>/dev/null | tr -d ' ')
  diff <(echo "$nums") <(echo "$nums" | sort -n) >/dev/null || echo "=== $dir/README.md rows out of order"
done
```

Note: BSD `awk` on macOS has no 3-arg `match()` — the `sed`-then-`awk` pipeline above avoids it.

For any gap found: add the missing row to the appropriate file and amend the last commit (`git commit --amend --no-edit`), or create a fixup commit.

This step must report **zero gaps** and no out-of-order rows before proceeding to push.

## Step 7 — Push and report

After all PRs processed:

```bash
git log --oneline master ^origin/master
```

Confirm list with user. Push:

```bash
git push origin master
```

GitHub auto-detects merged commits and closes the merged PRs. Verify:

```bash
gh pr list --state open --limit 50
```

Report final mapping table to user:

```
Closed (duplicates): #X, #Y
Merged (content updates): #A → BRC-NNN, ...
Merged (new BRCs): #P → BRC-MMM, ...
Remaining open: #Q (reason: draft / blocked / ...)
```

## Guardrails

- **Never** renumber a PR whose chosen number is free. Authors pick numbers deliberately (memetic, sequential with a sibling BRC, matching an external reference). A gap below it is not a reason; being above `LAST_BRC` is not a reason. Only a genuine collision is.
- **Always** ask the user before renumbering, even on a real collision — never silently remap.
- **Never** modify the substance of a contributor's document beyond:
  - BRC number reassignment (title heading + self-references only, after user approval)
  - Cross-reference rewrites for renumbered peers (Step 3)
  - Whitespace/conflict-marker cleanup
- **Always** confirm with user before closing a PR — closing is an external write under the user's identity.
- **Always** verify duplicate claim by diffing content, not just by title or BRC number.
- Pushing direct to `master` bypasses branch protection — user has authorised this for the docs repo. If push is rejected with a hard block, stop and ask.
- Use `TaskCreate` per PR to keep status visible; update as you go.
- If a PR's branch base predates a recent master refactor, the merge may pull in stale reverts. Inspect `git diff --cached --stat` before committing every merge — restore from `HEAD` any unrelated changes.
