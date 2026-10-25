# Alfie side chats: staging validation

October 25, 2026. No PRs merged and no app production deployment. Real staging UI evidence uses disposable Hollywood test identities and synthetic files only.

## Reviewed revisions

| Repository | PR | Revision |
|---|---|---|
| API | [#837](https://github.com/useallowance/allowance-api/pull/837) | `bc40c6f8f1456f918e566197fcab1cbb3b6a867e` |
| Web | [#347](https://github.com/useallowance/allowance-app-web/pull/347) | `dad011bb93452fe7ebe0012cac6d9d6021542acc` |
| iOS | [#441](https://github.com/useallowance/allowance-ios/pull/441) | `1e4cf5ebc77e78381714dba52ebacc876bf6772d` |

API #828 is closed. #837 is a fresh branch from origin/main with #820 included. Web and iOS retain the UI work and use #837's contract.

## What changed, and why it is lean

Main keeps its existing record. One small metadata table records side-chat ownership, title and hosted-session/workspace references. Message history stays in hosted sessions. Creating a side chat attaches a new independent session to Main's existing workspace; replacing a session uses #820's existing reuse/fresh-workspace fallback.

The API adds list/create/rename plus optional `chat_id` routing on existing chat operations. Existing callbacks, claims, notifications, schedules and Live delegation resolve the originating owned chat. Files and Calls remain account-owned. App-generated downloads use existing account Files IDs and content routes; no new Files storage, retry loop, worker, queue, cache, or transcript database was added. Archive/restore are absent from active code and UI.

API diff: 26 files, +998/-161. Production/SQL +468/-126; tests +502/-35; README +28. Most production changes are routes and existing session/callback ownership paths. Web: 11 files, +361/-34. iOS: 15 files, +400/-33.

Scope review found and fixed a non-clickable web entry, clipped rename menu, focus loss after rename, and native keyboard-dismiss/chat-button overlap. No remaining demonstrated code blocker was found. Missing live evidence is listed below rather than reported as a pass.

## Local validation

| Check | Result |
|---|---|
| API `uv run ruff check .` | Pass |
| API `uv run mypy .` | Pass, 487 source files |
| Full API suite | 4,376 passed; 55 skipped; 4 warnings |
| Local Postgres migration runner | Applied 0148; second run skipped correctly |
| Web production build | Pass, including final focus/menu changes |
| Web focused conversation/picker/shell/Files suite | 221 passed |
| Final web picker/shell checks | 30 passed |
| Final iOS APIClient/AppState/presentation tests | 79 passed |
| Signed Simulator core staging walkthrough | Pass |
| Signed Simulator file deletion/tombstone walkthrough | Pass |
| Signed Simulator drafts/rename/Calls walkthrough | Pass |
| Signed Simulator account file after source reset | Pass |

The temporary XCUITest target and OTP bridge were outside the product repositories. No product authentication hooks, deployment-target changes, or signing changes were committed. Initial harness problems included unsigned-build keychain persistence, duplicate accessibility matches, and iOS 26 confirmation popovers; these were corrected in the test setup/selectors. Failed harness attempts are not counted as passing smoke runs.

## Staging deployment

- API: deployed directly through `scripts/deploy_staging.sh` and Fly release command, serving the API revision above. No manual staging migrations.
- Web: deployed with local Vercel pull/build/prebuilt deploy to project `allowance-app-web-staging`, canonical URL [app-staging.useallowance.com](https://app-staging.useallowance.com/chat). Vercel's production target refers to this staging project.
- iOS: **Alfie Side Chats QA**, iOS **26.2**, UDID `1232DF8D-0309-4D73-B4CC-3642A98E2F7A`, connected to staging. No physical iPhone or TestFlight claim.
- API and iOS Actions checks could not start because GitHub reports failed account payments/spending limits. Direct staging deploys and local checks were used. The evidence gallery is a public GitHub Pages project and built successfully.

## Real staging coverage

| Area | Evidence and result |
|---|---|
| Main entry | Real coordinate click opened the web picker, including while Main worked. Main remains icon-only. |
| Picker | Pinned Main, selected state, new chat, empty state, loading, side title, rename-only menu. Desktop and 390x844 responsive layout. |
| Rename validation | Cancel, blank/whitespace disabled, trimmed title, Unicode and 100-character acceptance, 101-character rejection. Native rename retains its draft. |
| Independent chats | Main and side histories remain separate; web and native drafts stay with their chats. Native-created chat/reply appears on web. |
| Shared workspace | Main wrote a working-file marker and another side read the exact value. Read-only staging metadata confirmed distinct sessions sharing Main's environment after replacement. |
| Cold entry | Web reload and native cold launch return to Main. Chat list remains available. |
| Explicit targets | Valid chat target opens the requested side and is consumed. Unknown target is blocked; picker remains available to recover. |
| Ownership | 30 authorization/validation cases cover foreign, unknown and malformed IDs across history, session, stream, send, cancel, rename and reset, plus Main/invalid-title/stale-reset rules. |
| Generated Files | Real web- and Simulator-generated text files resolve to account Files URLs without session query parameters. Exact bytes verified. |
| Files across chats | Preview and Ask Alfie in Main and another side. Existing account file read on native. |
| Source replacement | Replaced web and Simulator source sessions; exact generated bytes and account previews remain available. Native reads the generated account file from Main afterward. |
| Multiple attachments | Two generated files retain order on web before/after reload and in native Main. Exact bytes verified separately. |
| Deleted Files | Native UI deletion, relaunch tombstone, no deleted download button, other attachment retained. Content returns 404. Repeated history reads do not resurrect the provider download. |
| Uploads | Real multipart side-chat upload accepted; account file bytes match, foreign access rejected, and content survives source-session replacement. This is API evidence, not a browser/native picker upload claim. |
| Busy reset | Provider was observed `in_progress`; manual reset rejected with the expected busy-session 409. |
| Cancellation | Observed busy side cancelled; retry with same cancellation ID accepted; independent Main turn completes; cancelled side becomes idle and can be replaced. |
| Calls | Native account Calls mini app opens normally. Unknown/foreign chat voice context denied. No call audio claim. |

[API contract assertions](api-contract-results.json), [generated Files](files-results.json), [source replacement](reset-results.json), [native source replacement](native-reset-results.json), [deletion](deletion-results.json), [busy cancellation](busy-cancel-results.json), [multipart upload](cancel-upload-results.json).

An initial upload/reset smoke assumed a submitted turn was already busy; the provider was still idle, so its reset correctly succeeded. The busy-reset test was rerun only after observing `in_progress`. Busy, idle and stale-reference behavior are recorded separately.

## Deterministic-only and missing live coverage

- Save-not-ready/failure and explicit Retry, unavailable-workspace fallback, Live delegation/session attribution, one-call guard, timezone preservation, notification routing and the 30-minute/Live-exception rules are covered by local tests. Storage faults were not injected into staging, and a 30-minute real-time background wait was not used.
- Scheduled is disabled on staging and returned the expected feature-gate 409. Runtime schedule execution was not tested.
- A real Live call, microphone/audio, and outgoing Phone call were not exercised. Account Calls UI and fixture-backed ownership/delegation checks are separate evidence.
- Browser control stalled at a deletion confirmation and the disposable QA browser session was lost. A fresh browser session resolved to the personal account; it was left unchanged. Web deletion completion, native/browser file-picker uploads, and a final manual keyboard-focus recheck are not claimed. File deletion was completed through native UI and verified through the shared API; uploads/cancellation were verified through staging API. Most web flows have screenshots from the successful QA session.
- The gallery was checked in a browser and its image URLs were checked for successful HTTP responses. Only synthetic QA content is published.

## Review and handoff

Temporary UI project/build, OTP bridge, QA auth files and credential-bearing setup traces were removed. Four passing Simulator result bundles and exported screenshots are retained locally. The signed app remains installed and running on the dedicated staging Simulator. Other worktrees and the other Simulator were preserved.

All three PRs remain open and unmerged. App staging deploys and Simulator evidence do not establish production or physical-device readiness. The remaining live checks above are explicit limits, not hidden passes.
