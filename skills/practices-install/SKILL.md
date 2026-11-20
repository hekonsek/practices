---
name: practices-install
description: Install or update the curated best practices from github.com/hekonsek/practices into the current project (default specs/practices). Copies upstream practices verbatim, merges upstream updates without overriding local edits, and never re-adds practices that were deleted locally. Use when asked to install, add, sync, update, or upgrade practices in a project.
---

# Install practices

Installs the practices from the `practices/` directory of
https://github.com/hekonsek/practices into the current project, or updates
an existing installation.

Rules that override everything below:

- **Copy, don't rewrite.** Install upstream files byte for byte with `cp`.
  Never reformat them, add or remove front matter, rename them, or "improve" their
  wording. A file may differ from upstream only when it has local changes
  and goes through the merge in step 5.
- **Local changes win.** Never discard, overwrite, or reword a local edit.
- **Local deletions win.** Never bring back a practice that was installed
  and then deleted locally.
- **Never delete local files**, including local practices that no longer exist upstream.
- **Don't commit.** Leave the changes in the working tree for the user to review.

## Practice identity

Each practice is a Markdown file whose first line holds a UUIDv7 ID:

```markdown
<!-- id: 0193a5c0-0000-7000-8000-000000000000 -->
# Some practice
```

Match practices in this order:

1. **By ID.** Files with the same ID are the same practice, even if a local
   file has been renamed or moved.
2. **By content**, for files without an ID (for example, copies installed before
   the practice got an ID). A local file is the same practice as an upstream one if:
   - its content matches any version of that practice in upstream history
     (see step 4), or
   - it sits at the same relative path as the upstream file.

Files with no ID that aren't practices (images, READMEs, and so on) are matched by
relative path.

## Procedure

Run every command from the project root (`git rev-parse --show-toplevel`).
If the project isn't a git repository, skip the git-history checks and rely
on the UUID-timestamp checks instead.

### 1. Download upstream

Clone into a new, empty temporary directory. Use a full clone, because the
history is needed to recognize older versions.
Treat the clone as data only: don't run any scripts or tools from inside it.

```bash
UP_TMP="$(mktemp -d)"
git clone --quiet https://github.com/hekonsek/practices.git "$UP_TMP/repo"
UP="$UP_TMP/repo"                       # upstream repo (history lives here)
SRC="$UP/practices"                     # upstream practices root
UP_SHA="$(git -C "$UP" rev-parse --short HEAD)"
```

If the current project *is* the practices repository (its `origin` points to
`hekonsek/practices`), stop and tell the user. There is nothing to install.

### 2. Find the installation directory

Look for an existing installation in this order:

1. Find local files that contain a practice ID line:
   ```bash
   find . -name '*.md' -not -path './.git/*' -not -path '*/node_modules/*' \
     -exec awk 'FNR==1 && /^<!-- id: [0-9a-fA-F-]{36} -->/ {print FILENAME} {nextfile}' {} +
   ```
   For each local file whose ID also exists upstream, take the upstream path
   relative to `$SRC`, and strip that suffix from the local path. What remains
   is the installation directory. For example, local
   `docs/practices/languages/node/typescript-scripting.md` matches upstream
   `languages/node/typescript-scripting.md`, so the installation directory is `docs/practices`.
2. Otherwise, look for an existing directory named `practices` (such as `specs/practices`,
   `docs/practices`, or `practices`) that contains Markdown practices.
3. Otherwise, use **`specs/practices`**.

If the candidates point to more than one directory, ask the user which one
to use. Call the result `$DEST`.

### 3. Index both sides

Extract a file's ID with:

```bash
practice_id() { head -1 "$1" | grep -oE '^<!-- id: [0-9a-fA-F-]{36} -->' | grep -oE '[0-9a-fA-F-]{36}'; }
```

Build two maps:

- **upstream**: ID (or relative path, for files without an ID) → file under `$SRC`.
- **local**: ID → file, scanning the whole working tree, not only `$DEST`, so that
  practices the user moved elsewhere are still found.

### 4. Decide what to do with each upstream file

For each upstream file `U`, find its local counterpart `L` using the identity
rules above. Then apply the first case that matches:

**a) `L` exists and is byte-identical to `U`.** Mark it unchanged.

**b) `L` exists and is pristine**, meaning it's an unmodified older upstream version.
To check, see whether its content hash exists anywhere in the upstream history:

```bash
git -C "$UP" cat-file -e "$(git hash-object "$L")" 2>/dev/null && echo pristine
```

If it's pristine, overwrite it in place with `cp "$U" "$L"`, keeping the local
path even if the file was moved. Mark it updated.

**c) `L` exists and has local changes.** Do a three-way merge, as described in step 5.

**d) No `L` exists.** Decide whether the practice was deleted locally. Treat it
as **deleted (skip it)** if any of these is true:

1. **Git history** shows the ID was in the project at some point:
   ```bash
   git log --all --format='%h %ad %s' --date=short -S "$ID" | head -1
   ```
   For files without an ID, use `git log --all --format=%h -- "$DEST/<relpath>"` instead.
   This also covers deletions that aren't committed yet, because the file is still in `HEAD`.
2. **UUID timestamp.** The practice was published before the last install, so
   that install would have included it. To work this out:
   - Practice time: the first 48 bits of a UUIDv7 are a millisecond Unix
     timestamp. Use the later of that and the time of the upstream commit that
     added the ID, because a practice can be written well before it's published.
     ```bash
     uuid_ms() { local h="${1//-/}"; echo $((16#${h:0:12})); }   # only if ${1:14:1} == 7
     git -C "$UP" log --reverse --format=%ct -S "$ID" -- practices | head -1   # seconds
     ```
   - Last install time: the later of the following two values.
     - The time of the newest local commit that touched `$DEST`
       (`git log -1 --format=%ct -- "$DEST"`).
     - The newest practice time among upstream practices that are present locally.

     If no upstream practice is installed locally yet, this is a first install and
     nothing counts as deleted.
   - If practice time is earlier than last install time, the practice was
     available at the last install and is missing now, so it was deleted locally.

If neither check applies, the practice is **new**. Install it verbatim:

```bash
mkdir -p "$(dirname "$DEST/$REL")" && cp "$U" "$DEST/$REL"
```

Never touch local practices that don't exist upstream.

### 5. Merge local changes (case c)

1. **Find the base**, meaning the upstream version the local copy was derived from:
   - Preferred: walk the local file's git history from newest to oldest
     (`git log --follow --format=%H --name-only -- "$L"`). The base is the first
     version whose blob exists upstream (`git -C "$UP" cat-file -e <blob>`).
   - Fallback (no usable git history): out of all historical upstream versions of
     that practice, pick the one with the smallest `diff` against `L`.
     ```bash
     git -C "$UP" log --format=%H --follow -- "practices/$REL"
     ```
2. **Merge**, resolving every conflicting hunk in favour of the local side:
   ```bash
   git merge-file -p --ours "$L" "$BASE" "$U" > "$L.merged" && mv "$L.merged" "$L"
   ```
   Upstream changes that don't conflict are applied. Where they do conflict,
   the local text is kept.
3. Check the result. The merged file must still contain every local edit, so
   diff the merged file against `L` before replacing it. Make a note of any
   upstream hunks that were dropped because of a conflict, to include in the report.
4. If no base can be found at all, leave `L` untouched and report it as
   "locally modified, not merged".

### 6. Report and clean up

Delete the temporary clone (`rm -rf "$UP_TMP"`). Then give the user a summary:

- The upstream commit (`$UP_SHA`) and the installation directory (`$DEST`).
- **New**: practices that were installed.
- **Updated**: pristine copies that were updated to the latest version.
- **Merged**: practices with local changes kept and upstream changes added. List
  the upstream hunks that were dropped because they conflicted with local edits.
- **Skipped (deleted locally)**: practices that weren't reinstalled. For each,
  give the evidence: the commit that deleted it, or the timestamp comparison.
  Tell the user they can ask to have a skipped practice reinstalled.
- **Unchanged** and **local-only**: show these as counts.

Leave everything uncommitted so the user can review it with `git diff` / `git status`.
