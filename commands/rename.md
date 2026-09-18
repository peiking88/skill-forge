---
description: Rename a skill — updates directory, all file references, workspace, and registry. Handles both project and user scope.
---

Rename a skill. `$ARGUMENTS` is `<old-name> <new-name>`.

All work is delegated to `rename_skill.py` so Claude never has to shell out
`mv`, Write `skill_registry.json` directly, or Edit files under
`<old>-workspace/` — each of those paths triggers a permission prompt
(unstable Bash allowlist / outside skill-dir trust exemption).

## Steps

1. **Parse args.** Extract `old-name` and `new-name` from `$ARGUMENTS`.
   If missing, ask the user with `AskUserQuestion` (load via
   `ToolSearch select:AskUserQuestion` if not in scope).

2. **Dry-run to build the plan.**

   ```bash
   python3 "${CLAUDE_PLUGIN_ROOT}/skills/skill-forge/scripts/rename_skill.py" \
     "<old-name>" "<new-name>" --dry-run
   ```

   The script auto-detects scope (user `~/.claude/skills/` first when it
   already holds `old-name`, else project `.claude/skills/` if present).
   Pass `--scope project` or `--scope user` to force.

3. **Review output.** The plan lists every file edit, directory rename, and
   the registry entry update. If it starts with `Errors (aborting):`, stop
   and report the errors — do NOT attempt manual workarounds.

4. **Ask for confirmation** with `AskUserQuestion`, showing the plan.
   Options: `Apply` / `Cancel`.

5. **Apply (on Apply)** — same command without `--dry-run`:
   ```bash
   python3 "${CLAUDE_PLUGIN_ROOT}/skills/skill-forge/scripts/rename_skill.py" \
     "<old-name>" "<new-name>"
   ```
   The script prints `Done. Renamed ...` on success. Relay that to the user.

6. **Publish-side sync (when the skill's source repo has a remote).** After
   the local rename succeeds, check for a remote:
   `git -C <skill-dir> remote get-url origin` (for symlink installs the
   skill dir resolves to the source repo). If one exists, list these for the
   user to confirm before running any of them:
   - `gh repo rename <new-name> --repo <owner>/<old-name>`
   - `git -C <skill-dir> remote set-url origin <new-url>`
   - Publish-address lines still referencing the old name (repo `CLAUDE.md`,
     README install commands)
   - Tag policy: pure rename → minor bump; a changed `name:` field (the
     invocation name) → ask the user whether it counts as breaking (major)

   Local install shape and remote publish shape are two independent planes —
   completing only the local half leaves the inconsistency for the next
   session.

## Notes

- The script guards against renaming while an active draft references the
  old name — it will error out and ask you to finish the current
  create/improve session first.
- Symlink installs (`~/.claude/skills/<name>` → a source repo) are
  supported: the link is repointed, and the source repo's files plus its
  repo-local registry (`.claude/skills/skill_registry.json` inside the repo)
  are edited in place. The source repo itself is never moved.
- Legacy `<old>-workspace/` sibling dirs (from before `.opt/` migration) are
  renamed too, with a warning suggesting manual migration into `<new>/.opt/`.
