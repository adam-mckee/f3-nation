# /fng onboarding: upstream plan

Working notes, not for merge. Source of the feature: private repo `adam-mckee/f3-fng-bot`
(see its `PORTING.md`). Written 2026-10-09 against upstream `main`.

Reviewer feedback so far (verbal): minimize the code changes as much as possible.

## Size if ported as-is

About 1,780 lines: `features/fng.py` alone is 702, the two test files about 500. It would
also add a new table, new settings fields, a new Slack scope and an hourly job. Too much
for one review.

## Biggest cut: write users through the database, not the REST API

The standalone bot creates users through `POST /v1/user`, which pulled in
`application/user/`, `infrastructure/api_client/user_repository.py` and an API key with
`editor` on the region. Upstream's slackbot already writes to Postgres directly via
`f3_data_models`:

- `utilities/helper_functions.py:create_user()` already creates `User` + `SlackUser` rows.
- `DbManager.create_record(User(...))` (`packages/db-python/f3_data_models/utils.py`) is the
  generic insert.

Inside upstream, the whole API-client user layer can likely become a few lines, and the
API-key-roles question disappears for reviewers. Keep the create-only rule: look the user
up by email first and never overwrite an existing user's roles.

## Split into three PRs

| PR | Adds | New infrastructure |
|---|---|---|
| 1. `/fng` signup | `/fng` form, confirm, create F3 profile, welcome-channel post, welcome email | None. Reuses `welcome_channel` and `downrange_invite_link`; email via `utilities/sendmail.send_via_sendgrid`; nothing stored because nothing happens later. |
| 2. Battle buddy loop | Call into `features/welcome.handle_team_join`; notify the buddy when the FNG joins | Onboarding table (schema in `packages/db` + model in `packages/db-python`), `users:read.email` scope |
| 3. Nudges | `send_fng_nudges` from `scripts/hourly_runner.py`; resend / stop / link-broken actions; admin alerts | `FNG_NUDGE_HOURS` / `FNG_NUDGE_TZ` on the scripts Cloud Run Job |

Split Slack messages by *when* they fire, not because they are Slack messages. The welcome
post is the payoff of `/fng`, so it stays in PR 1. Everything after signup (buddy notified,
nudges, broken-link alerts) needs storage and a scheduled job, which is exactly the heavy
part, so it goes in PRs 2 and 3. PR 1 then has no schema change and is useful on its own.

## Further trims for PR 1

- **No new settings.** Drop `fng_welcome_channel` / `fng_region_name`; use `welcome_channel`
  and `workspace_name` (the standalone bot's `fng_settings()` already falls back to them).
  A settings field means model + form + DB changes.
- **Match the nearest upstream feature's shape.** If comparable features (e.g.
  `features/welcome.py`) are one file, fold `application/fng/` (service, repository
  protocol, email text) into `features/fng.py` rather than adding a layered package.
- **Tests scoped to the PR.** Nudge and team-join tests move with PRs 2 and 3.
- **State the roadmap in the PR body:** "PR 1 of 3; storage and nudges follow", so the
  small scope reads as deliberate.

## Checks

From `apps/slackbot`: `uv run ruff check . && uv run mypy && uv run pytest`. The repo-wide
lefthook pre-commit gets OOM-killed on an 8 GB VM while `pnpm dev` runs; run checks
per-file and commit with `LEFTHOOK=0`.
