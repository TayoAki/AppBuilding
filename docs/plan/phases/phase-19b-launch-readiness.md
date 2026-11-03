# Phase 19b — Launch Readiness (Data Lifecycle, Template Upgrades, Vendor Watch, Gate G4)

| | |
|---|---|
| Video chapter(s) | New (not in video) |
| Branch | `feat/19b-launch-readiness` |
| Depends on | Phase 19a (production infra, observability), Phase 14 (export job for "Export my data"), Phase 18 (Stripe), Phase 00b fixture app |
| Effort (solo dev + Claude Code) | ~5–7 days |
| New env vars | none |

## Goal
Make the platform safe to open to the public, and keep it working as vendors move:
- people can delete their account and export their data, and retention rules run automatically (`08` §11);
- projects can move to a new Expo SDK without being regenerated (`06` §11, ADR-003 #9). This is needed **before the first SDK bump**, because store Expo Go supports only the current SDK;
- a nightly vendor watch catches upstream breakage before users do;
- gate **G4** (`14-confidence-gates.md` §2) passes, with evidence attached to the launch PR.

## What we do differently from the video (and why)
- The video ends at "deploy to Vercel". Real launches also need deletion and retention (legal), an upgrade path for the SDK the whole product depends on, and proof that security, load and restore work. These are the items that turn into incidents if skipped.

### Reference app notes
- The reference app's delete removes the project's rows and uploaded files but **not** its website, repo or backend, and the dialog says so. We keep that honesty for project and account deletion.
- The reference app pinned SDK 54 and had no upgrade path. Store Expo Go moved to 58, so its QR flow broke for current phones (`15` §1). The upgrade machinery here exists to stop that happening to us.

## Steps

### 1. Retention jobs (scheduled pg-boss jobs)
| Job | Rule (`08` §11) |
|---|---|
| `retention.tmp_uploads` | Bucket lifecycle rule (1 day); the job only verifies the rule exists |
| `retention.exports` | Delete export zips older than 7 days (storage + `runs.output.storageKey` nulled) |
| `retention.run_events` | Delete `run_events` 30 days after their run finished; keep the run summary |
| `retention.project_purge` | Hard-delete projects soft-deleted more than 30 days ago: rows, revisions, snapshots, assets, storage objects, thumbnails, device sessions |
| `retention.device_sessions` | Mark stale sessions stopped; delete rows older than 30 days |

### 2. Delete account (`runs.kind = account_delete`, single-shot, resumable)
- Settings → **Delete account** → re-auth → a confirm dialog listing what stays outside AppBuilding (Expo websites and builds, GitHub repos, backend projects), with links.
- Job steps, each idempotent:
  1. Cancel the Stripe subscription immediately and detach payment methods.
  2. Revoke provider tokens where an API exists (GitHub App user token, backend provider). Tell the user to revoke their Expo token. Delete `oauth_connections`. Offer the keystore download first if option B keystores exist, then delete `project_signing`.
  3. Hard-delete owned projects (same scope as the purge), memberships and pending invites. Shared projects owned by others stay.
  4. Anonymize `ai_usage` and `credit_ledger` (`user_id` → null). **Keep `signup_grants(email_hash)`** so a re-signup with the same email gets no second grant (ADR-003 #22).
  5. Delete the Better Auth user, sessions and accounts, then send a confirmation email.

### 3. Export my data
- Settings → **Export my data** → a job that reuses the Phase 14 zip export per project (`materialize({target:"export"})`) plus `profile.json`, `chat/<projectId>.json` and `ledger.csv`, all in one zip, with a signed link emailed (7-day expiry).

### 4. Template-upgrade machinery (`06` §11, ADR-003 #9)
- **Rule:** template **revisions** within one SDK render from HEAD for every project and must stay backward compatible, which CI checks by rendering the fixture and eval projects. Only **SDK changes** create a new versioned template.
- `templates/expo-app/versions/expo-<sdk>/` holds each SDK's template while any project uses it. `templates/migrations/<from>__<to>.ts` exports `migrate({plan, files}) → {plan, files, notes[]}`: deterministic codemods only, never the model.
- `upgradeProject(projectId, to)` (`runs.kind = template_upgrade`):
  1. run the migration chain in memory;
  2. validate every file (`05` §6);
  3. render-smoke-test every route in the **sandbox** (`07` §8);
  4. if both pass, write `author = "system"` revisions + a "Before upgrade to SDK N" snapshot, set `projects.template_version`, and emit `plan.updated`;
  5. if either fails, report the failures (with `notes[]` naming screens that need an AI change) and leave the project unchanged.
- UX: a workspace banner "This app uses SDK X. Upgrade to preview on the current Expo Go" → **Upgrade** (dry run, then apply) and **Undo** (restore the snapshot). Admin batch upgrade: `scripts/upgrade-projects.ts --from expo-58 --to expo-59 --dry-run`.
- **Runbook for a new SDK:**
  1. add the template version and its migration;
  2. run the eval suite on both versions;
  3. upgrade the fixture and eval projects;
  4. flip the default for new projects;
  5. batch-upgrade, oldest projects first;
  6. once `snack-sdk` supports the new SDK, re-check Phase 11 Path A.

### 5. Vendor watch
- **Nightly smoke** (`worker/handlers/vendor-smoke.ts`, results to a dashboard + alert on failure):
  - Device preview: start a Path B session for the fixture in the sandbox and fetch its manifest. Check whether the published `snack-sdk` now supports the template SDK, and log it for Path A.
  - EAS: `eas whoami` and project view with a test-account token; Hosting: a deploy of the fixture to a test project's preview alias.
  - ImageKit: fetch a known generated image; generate one new image with a salted name (weekly, to save units).
  - Backend provider: a health/contract test of the APIs we call (Supabase SQL endpoint is beta) against a test project.
  - Anthropic: one recorded-prompt plan call; check that structured output parses and the mid-conversation system message still works.
- **Monthly review** (calendar task): Expo, `snack-sdk`, EAS, ImageKit, provider and Anthropic release notes → update `15-vendor-facts.md` and start the upgrade runbook when an SDK lands.

### 6. Gate G4 evidence (attach to the launch PR)
- **Two-account pentest** per `08` §10:
  - two accounts try to read and modify each other's projects, runs, files, exports, device sessions, builds and backend data (gateway and direct modes);
  - XSS attempts from a generated screen;
  - an attempt to make generators write outside allowed paths;
  - forged Stripe and EAS webhooks.
- **Load test**: 20 concurrent builds using recorded model fixtures (no live spend) plus real sandbox renders, and 10 concurrent device-preview sessions. Record p50/p90 build time, outbox lag, sandbox memory and error rate.
- **Restore drill**: PITR restore into a scratch DB, a decrypt check with `ENCRYPTION_KEY`, keystore backup restore (option B), and a storage version restore.
- **Alerts firing**: spend, outbox lag, stuck steps, sandbox capacity (thresholds temporarily lowered).
- **Legal**: ToS (hosted content, acceptable use), privacy policy (model-provider processing, no training, retention table), cookie notice if needed. Footer links live.
- **Cost check**: the margin report from the last 7 days of beta usage is within targets (`11` §4), or prices are adjusted.

## Files to create / change
```
worker/handlers/{retention-*.ts,account-delete.ts,export-my-data.ts,template-upgrade.ts,vendor-smoke.ts}
templates/expo-app/versions/, templates/migrations/
lib/templates/{upgrade.ts,registry.ts}
scripts/upgrade-projects.ts
app/settings/{account,data}/page.tsx, app/api/account/{delete,export}/route.ts
components/workspace/UpgradeBanner.tsx
docs/runbook.md (+ SDK upgrade runbook, pentest checklist, restore drill)
docs/launch/g4-evidence.md (filled in for the launch PR)
```

## Database changes
- New `signup_grants (email_hash pk, granted_at)` (ADR-003 #22; created in Phase 02, retained here).
- `runs.kind` values `account_delete`, `template_upgrade`, `export_my_data` (add the last one to `03`).
- No other new tables. Uses `deleted_at`, `project_snapshots`, `projects.template_version`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/account/delete` | session + recent re-auth | → `{ runId }` |
| POST | `/api/account/export` | session | → `{ runId }` (link emailed) |
| POST | `/api/projects/:id/template-upgrade` | owner | `{ to, dryRun }` → `{ runId }` → `{ ok, notes[], failures[] }` in `runs.output` |
| POST | `/api/projects/:id/template-upgrade/undo` | owner | → `{ snapshotId }` |

## Acceptance checklist
- [ ] Retention jobs delete exactly what `08` §11 says (tests with time travel); a soft-deleted project is purged after 30 days, including storage.
- [ ] Account deletion on a test account removes or anonymizes everything listed in `08` §11; re-signup with the same email gets no second grant; the Stripe subscription is canceled.
- [ ] Export my data produces a zip with every project and the user's chat and ledger.
- [ ] Template upgrade on the fixture from SDK 58 to a test "59" template (a synthetic codemod) passes the dry run, applies, renders, and can be undone; a failing migration leaves the project untouched with clear notes.
- [ ] CI renders the fixture and eval projects against HEAD template revisions and fails on a backward-incompatible change.
- [ ] The nightly vendor smoke runs and alerts on a simulated failure.
- [ ] G4 evidence is complete in `docs/launch/g4-evidence.md`: pentest findings fixed or accepted, load-test numbers, restore drill, alerts, legal pages, cost check.

## Claude Code prompt
```text
We're implementing Phase 19b (launch readiness) of AppBuilding.

Read first:
- docs/plan/phases/phase-19b-launch-readiness.md (scope + acceptance checklist)
- docs/plan/08-security.md §10–§11 (pentest, data lifecycle)
- docs/plan/06-generated-app-template.md §11 and ADR-003 #9 (template upgrades)
- docs/plan/14-confidence-gates.md §2 (G4) and §4 (vendor watch)
- docs/plan/15-vendor-facts.md (what the nightly smoke must keep re-verifying)

Implement:
1. Retention jobs (tmp check, exports 7 d, run_events 30 d, project purge 30 d, device sessions).
2. Delete account job (re-auth; Stripe cancel; token revocation; owned-project hard delete;
   anonymize ai_usage/credit_ledger; keep signup_grants; delete auth rows; confirmation email).
3. Export my data job reusing the Phase 14 exporter.
4. Template-upgrade machinery: versioned SDK templates, deterministic migrations, upgradeProject
   (dry run → validate → sandbox render → apply + snapshot), banner, undo, batch script, and a CI
   job that renders fixture/eval projects against HEAD template revisions.
5. Nightly vendor smoke + alerts.
6. docs/launch/g4-evidence.md template and runbook sections (SDK upgrade, pentest, restore drill).
Tests: retention with time travel, account deletion end to end on a seeded account, upgrade dry-run
failure path, undo.
Do NOT change pricing, agents or phase 19a infra beyond what's listed.
Finish with lint, typecheck, tests; summarize, and list the G4 items I must run manually.
```

## Pitfalls & notes
- **The first SDK bump is a matter of when, not if.** Expo ships a new SDK a few times a year and store Expo Go follows within weeks. Have step 4 working before launch.
- **Deletion is irreversible.** Run account deletion on staging with production-like data before enabling it. Log each step's completion so a crashed job resumes instead of half-deleting.
- **Don't spend real money in load tests.** Use recorded model fixtures; the sandbox and DB load are what matter.
- **Keep the pentest honest.** Use a fresh session (or another person) that hasn't read the implementation, and attack only your own staging environment.
