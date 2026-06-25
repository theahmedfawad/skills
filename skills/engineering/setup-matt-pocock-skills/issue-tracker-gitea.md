# Issue tracker: Gitea

Issues and PRDs for this repo live on our self-hosted Gitea at **`http://192.168.20.60:3000`**. Use the [`tea`](https://gitea.com/gitea/tea) CLI for all operations. Authenticate once with `tea login add --url http://192.168.20.60:3000` (it stores a token); `tea` then infers the repo from `git remote -v` when run inside a clone, or pass `--repo <owner>/<name>` explicitly.

Exact flags vary by `tea` version — confirm with `tea <command> --help`. For anything `tea` doesn't expose (e.g. editing labels on an existing issue, or machine-readable comment threads), fall back to the **Gitea REST API** at `http://192.168.20.60:3000/api/v1/...` via `curl` with your token.

## Conventions

- **Create an issue**: `tea issues create --title "..." --body "..."` (use a heredoc for multi-line bodies; check `tea issues create --help` for `--labels`/`--description` naming on your version).
- **Read an issue**: `tea issue <number>` shows the issue; add comments via the API if `tea` doesn't print them (`GET /api/v1/repos/{owner}/{repo}/issues/{number}/comments`).
- **List issues**: `tea issues list --state open` with `--labels "..."` filters; use `--output csv`/`--fields ...` for parseable output (or the API: `GET /api/v1/repos/{owner}/{repo}/issues?state=open`).
- **Comment on an issue**: `tea issues comment <number> "..."` (or `tea comment <number> "..."`).
- **Apply / remove labels**: `tea labels` lists/creates labels; to change labels on an existing issue, use `tea issues edit <number> ...` if supported, else the API (`POST`/`DELETE /api/v1/repos/{owner}/{repo}/issues/{number}/labels`).
- **Close**: `tea issues edit <number> --state closed` (or the API). Post the explanation as a comment first if the close path can't carry one.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

When set to `yes`, PRs run through the same labels and states as issues, using the `tea pr` equivalents:

- **Read a PR**: `tea pr <number>` and `tea pr` for the list; fetch the diff via the API (`GET /api/v1/repos/{owner}/{repo}/pulls/{number}.diff`) if `tea` has no diff subcommand on your version.
- **List external PRs for triage**: `tea pr list`, then keep only PRs authored by non-members (a contributor's PR, not a maintainer's in-flight work) — check the author against the repo's collaborators via the API if needed.
- **Comment / label / close**: `tea pr comment` (or `tea comment`), label via the API, `tea pr close <number>`.

Like GitHub, Gitea shares one number space across issues and PRs, so a bare `#42` may be either — resolve with `tea pr 42` and fall back to `tea issue 42`.

## When a skill says "publish to the issue tracker"

Create a Gitea issue.

## When a skill says "fetch the relevant ticket"

Run `tea issue <number>`.
