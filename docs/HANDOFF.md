# Handoff — next steps for a new Claude Code session

State as of 2026-11-03.
- **Built:** Milestones 1b–4 are built and tested (174 tests).
- **Waiting on API keys:** the library build and the real search-quality test set (golden set).

## Approvals already given by the project owner
- **Library build:** draft **40 cards**, total spend capped at **$80** (≤ $2 per card). Pass the cap explicitly
  (`LIBRARY_BUILD_BUDGET_USD=80`); the default is 0, which refuses to spend.
- **Golden set:** the build team drafts ~30 reporter-style problem statements against real cards, marked
  `status: proposed`; Knight Lab editors later mark them `confirmed`. CI gates only on confirmed pairs.

## Keys (from the environment's secrets; never ask for them in chat)
- `SOLBRIEF_ANTHROPIC_API_KEY`: the drafter, the verifier's entailment check and the classifier fallback. Use
  this name, not `ANTHROPIC_API_KEY`, which Claude Code itself would also pick up.
- `VOYAGE_API_KEY`: embeddings.
- `OPENALEX_API_KEY`: optional.

Check they're present without printing them, e.g.
`for k in SOLBRIEF_ANTHROPIC_API_KEY VOYAGE_API_KEY; do [ -n "${!k}" ] && echo "$k set"; done`.

## 1. Local Postgres + pgvector (cloud containers start without it)
```sh
apt-get update -qq && apt-get install -y -qq postgresql-16-pgvector   # Postgres 16 is preinstalled
export PGDATA=/tmp/pgdata
su postgres -c "/usr/lib/postgresql/16/bin/initdb -D $PGDATA -A trust" >/dev/null
su postgres -c "/usr/lib/postgresql/16/bin/pg_ctl -D $PGDATA -l /tmp/pg.log -o '-p 5433 -k /tmp' start"
psql -h /tmp -p 5433 -U postgres -c "create database solbrief_dev"
uv venv -p 3.12 .venv && uv sync --frozen --extra dev --extra agents
TEST_DATABASE_URL="postgresql://postgres@/postgres?host=/tmp&port=5433" uv run pytest   # expect all green
```
Note: the container can restart and stop Postgres. If you get "connection refused", remove
`$PGDATA/postmaster.pid` and run `pg_ctl ... start` again.

## 2. Build the library (≤ $80)
```sh
export DATABASE_URL="postgresql://postgres@/solbrief_dev?host=/tmp&port=5433" APP_ENV=dev LIBRARY_BUILD_BUDGET_USD=80
uv run alembic upgrade head
uv run solbrief library-build --cards 40      # drafts → verifies (with entailment) → embeds; prints spend report
# inspect results: run the web app and GET /admin/cards (Bearer $ADMIN_TOKEN)
```
Report to the owner: spend, cards drafted / verified / provisional / not publishable, and the top drop reasons
(from `GET /admin/cards` or `solution_cards.verification`).

## 3. Golden set + matcher weights
1. Rewrite `evals/matcher_golden/golden.yaml` against the real publishable cards: about 30 pairs spanning the
   classes, mixing local (with `fips`) and national framings, all `status: proposed`. Set
   `library_kind: live`.
2. Run `uv run solbrief golden-snapshot`, which freezes the cards and query embeddings into `library.json`.
3. Run `uv run solbrief matcher-fit` to tune `src/solbrief/matcher/weights.yaml`. Record recall@5 before and
   after.
4. Leave `threshold_confirmed_recall_at_5: null` until editors confirm pairs. Then set it to the confirmed-pair
   baseline (minus a small margin).
5. Run the full test suite, then commit and push.

## Still open (owner actions)
- **A2:** Fly staging setup (`docs/DEPLOY.md`) and the `FLY_API_TOKEN_STAGING` GitHub secret.
- **A5:** the SJN partnership request.
- **A7:** Knight Lab curators to confirm golden pairs.
- **A8:** a test inbox list for Milestone 5 (writer, evaluator, email via Postmark).
