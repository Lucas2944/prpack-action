# prpack-action

> Pack each pull request into one markdown file optimized for LLM code review.

A GitHub Action wrapper around [prpack](https://github.com/Lucas2944/prpack). On every PR, it builds a single markdown file containing the diff *and* the full post-change content of every touched file, uploads it as an artifact, and leaves a summary comment. Drop the artifact into Claude / Cursor / your model of choice and ask for a review that can actually see what didn't change.

The technique behind it is written up at length here: [Your LLM code reviewer is reading half the file](https://scottthurman.hashnode.dev/your-llm-code-reviewer-is-reading-half-the-file).

## Usage

The minimum:

```yaml
# .github/workflows/prpack.yml
name: prpack

on:
  pull_request:

permissions:
  contents: read
  pull-requests: write

jobs:
  pack:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: Lucas2944/prpack-action@v1
```

That's it. On every PR, you'll get:

- An artifact called `prpack-context` containing the packed markdown
- A summary comment on the PR with file count and token estimate

Open the artifact, paste it into Claude or Cursor, and ask "Review this PR."

## Inputs

| Input | Default | Description |
|---|---|---|
| `base` | PR base, or `origin/main` | Base ref for the diff. |
| `head` | `${{ github.sha }}` | Head ref. |
| `config` | — | Path in your repo to a `.prpack.yml` config file. |
| `include-tests` | `false` | Pull in adjacent test files even if they didn't change. |
| `exclude` | — | Newline-separated globs to exclude. |
| `max-bytes` | `200000` | Skip individual files larger than this. |
| `no-content` | `false` | Diff-only output, no full file content (smaller pack). |
| `output-path` | `prpack-context.md` | Where to write the packed file. |
| `upload-artifact` | `true` | Upload the packed file as a workflow artifact. |
| `post-summary-comment` | `true` | Post a summary comment on the PR. |

## Outputs

| Output | Description |
|---|---|
| `output-path` | Path to the packed markdown. |
| `size-bytes` | Size of the packed markdown in bytes. |
| `files-changed` | Number of files included in the pack. |

## Recipes

**Skip the comment, just upload the artifact.**

```yaml
- uses: Lucas2944/prpack-action@v1
  with:
    post-summary-comment: 'false'
```

**Diff-only for huge PRs.**

```yaml
- uses: Lucas2944/prpack-action@v1
  with:
    no-content: 'true'
```

**Use your own preset.**

Drop a `.prpack/security.yml` (or wherever you like) into your repo:

```yaml
preface: |
  This PR touches our auth flow. Focus the review on input validation,
  authorization checks, and token handling.
reviewPrompt: |
  You are a security reviewer. For every issue you find, give file:line,
  severity (CRITICAL / HIGH / MEDIUM / LOW), why it matters, and a
  concrete fix.
```

Then point the action at it:

```yaml
- uses: Lucas2944/prpack-action@v1
  with:
    config: .prpack/security.yml
    include-tests: 'true'
```

**Exclude generated paths.**

```yaml
- uses: Lucas2944/prpack-action@v1
  with:
    exclude: |
      pnpm-lock.yaml
      dist/**
      **/*.snap
```

## Why

Pasting a raw diff into a model gets you generic feedback. The model can't see the rest of the file, so it pattern-matches on the diff and says "looks good" to changes that broke things outside the diff. Adding the full post-change content of every touched file is a one-line fix that turns useless reviews into reviews that catch real bugs. This action does it for you on every PR.

## Permissions

The action needs `contents: read` to run `git diff`. It needs `pull-requests: write` only if `post-summary-comment` is `true` (the default). For PRs from forks, GitHub-issued tokens are read-only by default — set the workflow to `pull_request_target` and trust accordingly, or disable comments.

## How it works

It's a composite action. Internally it runs:

```sh
npx -y github:Lucas2944/prpack --base <base> --head <head> --out prpack-context.md ...
```

Then uploads the file with `actions/upload-artifact@v4` and posts the summary comment with `gh pr comment`. No new dependencies in your repo, no Docker container.

## Curated review presets

For four ready-made review-style presets (security / performance / tests / architecture) plus a one-page workflow guide on getting useful reviews out of LLMs, see the [prpack Pro Pack](https://scottthurman89.itch.io/prpack) — free or pay-what-you-want. The action stays MIT.

## License

MIT.
