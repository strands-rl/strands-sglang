Review the open Dependabot PRs and, if they are safe, close them in favour of a single hand-made PR that lands the same bumps.

`$ARGUMENTS` optionally lists PR numbers (e.g. `100 101`). Without it, take every open PR authored by `app/dependabot`. Never mix ecosystems in one PR: `uv` bumps (prefix `build:`) and `github-actions` bumps (prefix `ci:`) get one consolidated PR each.

## 1. Collect

```bash
gh pr list --author app/dependabot --state open --json number,title,headRefName,statusCheckRollup
gh pr diff <n>            # per PR
gh pr view <n> -q .body   # release notes, compatibility score
```

Also read `.github/dependabot.yml` (cooldown days, groups, ignore list) and the `uv-pre-commit` rev comment in `.pre-commit-config.yaml`.

## 2. Judge

A PR is okay to consolidate when all of these hold:

- Every CI check passed on the PR.
- `uv` PRs touch only `uv.lock`; `github-actions` PRs touch only `.github/workflows/*.yml`. A `pyproject.toml` change means a floor moved — review it as code, not as a bump.
- Minor or patch bump. For a major bump, or release notes mentioning a removal, rename or default change in an API this package or its tests use (`transformers` chat templates and `tokenizer.encode`, `strands-agents` hooks/model interface, `pytest`/`ruff`/`mypy` config keys), stop and report to the user with the relevant notes quoted. Do not consolidate it.
- The dependency is not in the `ignore:` list (e.g. `e2b` is pinned on purpose).

Report the verdict per PR before changing anything. Leave a not-okay PR open and untouched; only the okay ones go into the consolidated PR.

## 3. Branch and reproduce the change

```bash
git fetch origin && git checkout -b build/bump-<summary> origin/main
```

**`uv` ecosystem — regenerate, do not copy.** Use the uv version pinned for the `uv-lock` pre-commit hook; a different uv rewrites unrelated dependency markers (seen with `secretstorage`). Pin each package to the exact version Dependabot chose: a plain `--upgrade-package foo` grabs the newest release and skips the cooldown in `dependabot.yml`.

```bash
uvx --from 'uv==<pre-commit rev>' uv lock --upgrade-package 'foo==X.Y.Z' --upgrade-package 'bar==A.B.C'
```

Transitive bumps in the Dependabot diff (e.g. `tokenizers` under `transformers`) come along on their own; do not name them.

**`github-actions` ecosystem — apply the diffs.** `gh pr diff <n> | git apply` per PR. Keep the `# vX.Y.Z` comment next to each SHA in sync.

**Prove equivalence.** Apply every included PR diff onto a scratch copy of `origin/main` and `diff` it against the working tree. It must be byte-identical; anything else means a version, a marker or a uv version is off. Fix the cause, never hand-edit the lock.

```bash
TMP=$(mktemp -d) && git show origin/main:uv.lock > "$TMP/uv.lock"
(cd "$TMP" && git init -q && git add . && git -c user.name=x -c user.email=x@x commit -qm base \
  && for n in <numbers>; do gh pr diff "$n" | git apply; done)
diff -q "$TMP/uv.lock" uv.lock && echo IDENTICAL
```

## 4. Verify together

The Dependabot PRs were each tested alone; this is the first run of the combination.

```bash
uv sync --locked
uv run pre-commit run --all-files --show-diff-on-failure   # what CI's lint job runs; includes uv-lock, mypy, pytest (fast)
uv run pytest tests/unit -q
uv run ruff check src/ tests/ examples/ && uv run ruff format --check src/ tests/ examples/   # the venv's new ruff, not the hook's
```

Any failure ends the consolidation: report it with the output and leave the Dependabot PRs open.

## 5. Commit, PR, close

Commit message: conventional prefix, one line per bump in the body, which PRs it supersedes, which uv generated the lock and why, what was verified. Add `Co-Authored-By: Claude <noreply@anthropic.com>`.

```bash
git push -u origin <branch>
gh pr create --base main --title "<type>: bump <pkgs> ..." --body-file - <<'EOF'
Supersedes #<n> and #<m> ...
| Package | From | To | Group |
...
## Verification
...
EOF
```

Then close each included Dependabot PR with a comment naming the new PR: `gh pr close <n> --comment "Superseded by #<new>, ..."`. Closing (not `@dependabot ignore`) is right: once the new PR merges the lock is at those versions, so Dependabot has nothing to re-open.

Wait for CI on the new PR (`gh pr checks <new>`) and report the result. Do not merge; that is the user's call.

## 6. After the user reports the merge

```bash
git checkout main && git pull --ff-only origin main
git branch -d <branch>     # "not yet merged to HEAD" is expected after a squash merge
git fetch --prune origin
uv lock --check
```

GitHub deletes the remote branch on merge; if `git ls-remote --heads origin <branch>` still shows it, `git push origin --delete <branch>`.
