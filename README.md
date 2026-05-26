# AntiDiffamine

Visualize the effect of a force-push. Given two commits (`source` → `target`) and a base branch, AntiDiffamine rebases the source commit range onto the target and renders the result as a GitHub-style diff — with range-diff, merge conflict highlighting, and an optional hosted HTML page.

---

## 1. Try the demo

Open an issue in this repo and post a comment using the slash command:

```
/antidiffamine repo=<owner>/<repo> source=<sha-before> target=<sha-after> head_branch=<branch>
```

**Example** — compare two commits on the Kubernetes repo:

A hetero-ancestral pair without merge conflicts:
```
/antidiffamine repo=kubernetes/kubernetes source=cfb9303d9708f9a2e67b2b2960cc23aa04af7574 target=8058052052ec2729ac564b0130affedd0cc63533 head_branch=master
```

A hetero-ancestral pair with merge conflicts:
```
/antidiffamine repo=kubernetes/kubernetes source=94ec47a849e4a21b1bbf4649314c69daea86f018 target=a35dc76581afb8cd8567ff357531c1deab353460 head_branch=master
```

A hetero-ancestral pair that AntiDiffamine fails:
```
/antidiffamine repo=kubernetes/kubernetes source=7033f14056de4b38d78c123f0f96a7fd286565c6 target=8eac3f9d310cc5c794b0ad5c3a15bf8ead0237c5 head_branch=master
```

The bot will react 👀 to acknowledge, then reply with:
- A link to the live **Actions run** (inline diff available immediately in the run summary)
- A link to the **hosted HTML page** on GitHub Pages (ready in ~2–5 min)

> **Where to find the SHAs on a GitHub PR:** open the PR, click *"force-pushed"* in the activity timeline — GitHub shows `from <sha> to <sha>`. The head branch is listed at the top of the PR next to the branch name.

---

## 2. Add it to your own repo

Drop [`examples/antidiffamine.yml`](examples/antidiffamine.yml) into your repo at `.github/workflows/antidiffamine.yml`. No secrets needed.

> **Permissions note:** Two settings are required in your repo (**Settings → Actions → General**):
> 1. **Workflow permissions → Read and write permissions** — lets the workflow push to `gh-pages`.
> 2. **Allow GitHub Actions to create and approve pull requests** — lets the workflow post PR comments.
>
> `secrets: inherit` is required on the `antidiffamine` job so the caller's `GITHUB_TOKEN` (not the AntiDiffamine repo's token) is used inside the reusable workflow when pushing to `gh-pages`.

```yaml
# .github/workflows/antidiffamine.yml
name: AntiDiffamine — PR force-push diff

on:
  pull_request:
    types: [synchronize]

jobs:
  antidiffamine:
    if: github.event.before != github.event.after
    uses: antidiffamine/AntiDiffamine/.github/workflows/antidiffamine-demo.yml@main
    with:
      repo: ${{ github.repository }}
      source: ${{ github.event.before }}
      target: ${{ github.event.after }}
      head_branch: ${{ github.base_ref }}
    permissions:
      contents: write

  comment-result:
    needs: antidiffamine
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            # posts a comment on the PR with links to the diff
```

Whenever someone force-pushes to a PR branch, the workflow posts a comment like:

> 🔁 **Force-push detected** — here's what changed:
>
> - 📄 [Rendered diff (GitHub Pages)](https://you.github.io/your-repo/runs/123/) — live in ~2–5 min
> - 📋 [Inline diff (Actions summary)](https://github.com/you/your-repo/actions/runs/123) — available now
>
> Compares `a60ee11` → `a325a42` relative to `main`

### HTML rendering (optional)

If you don't need the hosted HTML page, set `html_enabled: false` in the `with:` block. This skips rendering `diff.html`, uploading the artifact, and deploying to Pages. The inline Actions summary is still generated and available immediately after the job completes.

### GitHub Pages (optional)

To get a hosted HTML visualization linked in the PR comment, set up Pages in three steps:

1. Create a `gh-pages` branch in your repo (it can be empty):
   ```bash
   git switch --orphan gh-pages && git commit --allow-empty -m "init gh-pages" && git push origin gh-pages
   ```
2. Go to **Settings → Pages → Build and deployment**, set **Source** to **Deploy from a branch**, and select the `gh-pages` branch with `/ (root)`.
3. Ensure the workflow has write access: the `permissions: contents: write` key in `antidiffamine.yml` covers this.

Once configured, each diff run pushes a standalone `index.html` to `gh-pages` and the PR comment includes a direct link to it. If Pages is not set up, this step is skipped automatically and only the inline Actions summary link is posted — no configuration needed to use the basic diff.

#### Cleaning up old pages

Each diff run pushes an `index.html` to your `gh-pages` branch under `runs/<run-id>/`. These accumulate over time — **you must add the cleanup workflow** to remove them, otherwise your `gh-pages` branch grows unbounded.

Copy [`.github/workflows/cleanup-pages.yml`](.github/workflows/cleanup-pages.yml) into your repo at the same path:

```
.github/workflows/cleanup-pages.yml
```

This runs daily and deletes any diff older than **3 days**. Without it, old diffs stay on your `gh-pages` branch and Pages storage indefinitely.
