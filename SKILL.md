---
name: git-resolve-conflicts
description: rebase/merge で発生したコンフリクトをファイル種別ごとの方針（設定ファイルは意図を汲んで手動マージ、自動生成ファイルは再生成コマンド任せ）で解決する。git-rebase / git-merge から呼ばれるほか、単体でも、GitHub PR URL を渡しても起動できる。ユーザーが「コンフリクト直して」「PRのコンフリクトを解決して」など、マージ/リベースの衝突解決を求めている場合は必ずこのスキルを使う。
---

# Skill: git-resolve-conflicts

## Arguments

- No arguments: resolve conflicts from a rebase/merge currently in progress, in place (existing behavior).
- A GitHub PR URL (`https://github.com/<owner>/<repo>/pull/<number>`):
  1. Get the target (head) and base branches with `gh pr view <url> --json number,headRefName,baseRefName`.
  2. Update both branches with `git fetch origin <baseRefName> <headRefName>` (fetching is allowed for this PR-triggered call — this is a separate rule from the "don't fetch" rule that applies when git-rebase/git-merge are invoked standalone).
  3. Check out the target branch locally (`gh pr checkout <number>` if it doesn't exist locally yet; otherwise `git switch <headRefName>` then `git reset --hard origin/<headRefName>` to bring it up to date).
  4. Run `git rebase origin/<baseRefName>`.
  5. If conflicts occur, resolve them following the "Steps" section below.
  6. After resolving, report to the user per "Report format", and **post the same content to the PR with `gh pr comment <number> --body-file <temp-file>`**.
  7. Pushing the rewritten branch back to origin (force push) always requires the user's explicit permission first. Never run `git push --force` / `--force-with-lease` without asking.

## Preconditions

- For a no-argument call, check `git status` to determine whether it's `rebase in progress` or `You are currently merging`. The `--continue` step below follows this determination.
- If there are no conflicts, just report "No conflicts" and finish.

## Report format

For each conflict-resolved file, report using the following format (same format for PR comments):

```
## <file name> <line numbers>

\`\`\`diff
<relevant diff after resolution>
\`\`\`

<brief note on the cause of the conflict, how it was resolved, and the scope of impact>
```

- Give `<file name> <line numbers>` its own heading per resolved spot (multiple headings if a single file has multiple conflict spots).
- The diff block should show the diff (or the resolved hunk) so before/after is clear.
- The body should briefly touch on "cause of conflict," "resolution approach," and "scope of impact" (no sub-headings needed, plain prose is fine).

## Steps

1. List conflicted files from `git status` (`UU`, `AA`, `AU`, `UA`, etc.).
2. Classify each file by type and branch the approach:

   - **Human-authored config files (e.g. `package.json`)**: Read the diff containing conflict markers with `git diff` to understand both sides' intent. Don't simply pick one side — merge in a way that honors both sides' intent (e.g., if one side adds a dependency and the other bumps a version, reflect both). **Always summarize the resolution to the user** (never resolve silently). If the intents genuinely conflict, ask the user.
   - **Auto-generated files** (not limited to lockfiles like `yarn.lock` / `package-lock.json` / `pnpm-lock.yaml` — this covers build artifacts, generated code, and any file people don't hand-edit): **manual resolution is forbidden**.
     - Resolve the conflict in the corresponding hand-written file first (e.g. `package.json`), then run the regeneration command (`yarn install` / `npm install` / `pnpm install`, etc. — determine which from the project's `packageManager` field or the file type) and let it regenerate automatically.
     - If the regeneration method is unclear, don't guess — **ask the user**. Once you learn the regeneration method, save it to memory so you don't have to ask again next time.
     - After running the regeneration command, check `git status` for any newly untracked files or new diffs, and `git add` anything found (don't miss related files updated as a side effect of regeneration).
   - **Other text files (code, etc.)**: Read the context around the conflict markers and resolve based on both sides' intent. If the intents conflict and it's not obvious, ask the user.

3. For all target files, confirm no conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) remain.
   ```bash
   grep -rn '^<<<<<<<\|^=======$\|^>>>>>>>' <resolved files>
   ```
4. `git add <file>` only the resolved files (don't sweep in unrelated unstaged changes).
5. Continue based on the state determined in Preconditions.
   - Mid-rebase: `git rebase --continue`
   - Mid-merge: `git commit` (create the merge commit; leave the default message as-is in general).
6. If more conflicts appear, go back to step 1. Once done, check the final state with `git status` / `git log --oneline -5` and report.

## Rules (never violate)

- Resolving auto-generated files (lockfiles, etc.) by hand-editing.
- Guessing at how to regenerate an auto-generated file when the method is unclear, instead of asking.
- Running `git add` while conflict markers remain.
- Resolving semantic conflicts in files like `package.json` without reporting to the user.
- Aborting a rebase/merge (`--abort`) without the user's explicit instruction.

## Output

- Report each file's resolution approach concisely, per "Report format".
- If regenerating an auto-generated file caused additional files/diffs to be `git add`ed, include that in the report (what was added and why).
- Present the final `git status` and recent commit log.
- If an unpushed branch was rewritten by rebase, warn that pushing will require a force push, and get permission before doing so.
- For PR-URL-triggered calls, also post the above report to the PR via `gh pr comment`.
