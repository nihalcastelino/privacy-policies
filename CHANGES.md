# MedicalOS privacy update: draft changes (branch `privacy-update-2026-10`)

DRAFT for the owner's review. Not pushed, not published. Second pass: all `[OWNER: ...]` markers that evidence could settle are
settled (section 4); eight markers remain in the HTML, all the one publication-date token; see section 4.4. Delete this file before merging to `main`
(GitHub Pages would publish it).

Code references are paths in `MedicalOS` at master `8985a61` (clean checkout `MedicalOS-store-release-8985a61`).

Files changed: `privacy-policy-medicalos.html` (rewritten), `delete-account-medicalos.html` (rewritten),
`terms-medicalos.html` (6 small edits), `support-medicalos.html` (2 Q&As, dates). Same CSS, same page structure,
no new classes; owner markers use `<mark>` so they show highlighted.

## 1. Substantive changes against the live policy (25 August 2026)

| # | Live policy said | Draft says | Evidence in code |
|---|---|---|---|
| 1 | "applies to the mobile applications and the web application" | Also covers MedicalOS for Windows (Microsoft Store and GitHub installer) | `apps/desktop/src-tauri/tauri.conf.json`, `docs/desktop/T6-CONTRACT.md` |
| 2 | "collects your email address, an account identifier and your study progress. That is all." | Replaced; the full list is section 2 | below |
| 3 | "No push notification tokens. The app sends no push notifications." | Push tokens collected (optional), generic notifications, Expo/Apple/Google as processors | `apps/api/src/routes/push.ts` (`push_devices`: token, platform, `device_name`, active, last_seen); `supabase/migrations/026`, `039`; `supabase/functions/push-dispatch/dispatch.ts` (Expo); `apps/api/src/services/live-push.ts` ("Lecture started"); `apps/mobile/src/lib/push-notifications.ts:139` (`Device.deviceName`). Notification text is generic: `093_class_messages_without_lesson.sql:97` ("New faculty message") |
| 4 | not mentioned | Signed-in devices: random device key, platform, device name (phone model, Windows `COMPUTERNAME`, browser description), first/last seen, removal reason; device limit (two) | `085_account_devices.sql`, `086_desktop_platforms.sql`, `apps/api/src/services/device-limit.ts`, `packages/shared/src/device-limit.ts`, `apps/mobile/src/lib/device-headers.ts`, `apps/desktop/src-tauri/src/identity.rs` |
| 5 | not mentioned | App installation record (install id, platform, app/build version, update id, channel, first/last seen) | `037_app_releases.sql`, `090_store_channel.sql`, `apps/web/src/lib/desktop-heartbeat.ts` |
| 6 | not mentioned | Screenshot reports (kind, platform, screen route, time, device; never the image) and the on-screen watermark | `085` (`screen_capture_events`), `apps/api/src/routes/account-devices.ts:162`, `docs/DEVICE-LIMIT.md` |
| 7 | not mentioned | Classes, join requests, institution membership, email class invitations (the invited email is stored), lecture attendance, questions, poll answers | `026`, `042`, `046`, `088`, `089`, `091`, `092`; `021`, `055` |
| 8 | not mentioned | Inbox, lecturer messages, notification preferences | `038`, `039`; `apps/api/src/routes/push.ts` |
| 9 | "AI study features on the web application ... We do not store AI conversation transcripts." | AI marks typed answers on all apps (cases and lesson checks). Answers are sent to Anthropic without identity. Answers and feedback ARE stored. AI usage records and a monthly spend counter. No chat. | `apps/api/src/services/ai-grader.ts`, `routes/cases.ts:79`, `routes/ai.ts`, `services/ai-audit.ts` (hashes only), `services/ai-budget.ts` (Redis, 40-day TTL), `081_evidence_integrity.sql:58` (`raw_submission` = the typed answer), mobile calls `/cases/:id/steps/:id/attempt` and `/curriculum/lessons/:id/check` |
| 10 | Processors: Supabase, Anthropic, Netlify, Railway | Adds Upstash, OpenAI (lecturer figure text only), Expo, Apple/Google, Microsoft, GitHub, email provider; region column | `apps/api/src/middleware/redis.ts`, `routes/cms-images.ts` (`requireFaculty`), `apps/desktop/src-tauri/src/feed.rs` (GitHub update feed) |
| 11 | "readable only by your own account" | New section 5 "Who can see your data": lecturers see rosters, attendance CSV (name, email, times), and answers/marking for their own topics; admins see the account list, devices, screenshot reports, AI costs | `routes/classes.ts:341`, `services/live-sessions.ts` (`loadLectureRoster`), `routes/live.ts` roster export, `053_privacy_surfaces.sql:7` (`privacy_attempt_teacher`), `routes/faculty.ts:345-375`, `routes/admin.ts:25`, `routes/account-devices.ts` |
| 12 | not mentioned | Logs/IP, rate limiting, update checks (Expo, GitHub), what stays on each device, Windows Credential Manager entries survive uninstall | `build-app.ts:80` (`logger: true, trustProxy: true`), `middleware/redis.ts`, `apps/mobile/app.config.ts` (`updates.url`), `apps/desktop/src-tauri/src/secrets.rs`, `auth_store.rs`, `docs/desktop/T6-CONTRACT.md` rows 5, 6, 10 |
| 13 | Retention: one paragraph | Per-category table (honest: kept until account deletion; no inactivity deletion exists) | see gaps 5-6 |
| 14 | Rights: access, correct, delete | Adds DPDPA nominate-another-person and Board complaint; deletion section 11 | none in code; statute wording for counsel to confirm |
| 15 | Delete page: faculty get an "anonymised, disabled stub" | Removed: the route never creates a stub; the auth user is always deleted | `apps/api/src/routes/account.ts` header ("the route never creates a disabled identity stub"), `045_privacy_retention.sql` |
| 16 | Delete page listed 5 data groups | 11 groups, plus a "what is kept, and why" section and the Windows Credential Manager note | `account.ts` `PERSONAL_DATA_TABLES`, FK cascades, `091` section 5 |
| 17 | Terms: nothing on shared logins | Account is personal; device limit; watermark; no bypassing protections; downloaded lessons removed on sign-out | `docs/DEVICE-LIMIT.md` |
| 18 | Support | Device-limit Q&A; "remove everything from my Windows PC" Q&A | |

Kept as is: contact `nihal@cloudzzy.co.uk`, DPDPA framing (now alongside UK GDPR), 30-day response,
18+ / not directed at children, Apple Hide My Email paragraph, backups paragraph (now with a period).

Second pass (2026-10-15): the controller is CLOUDZZY LTD, a UK company (details from the owner), so the policy was
rewritten for UK GDPR + DPDPA; see section 4.

Age: Play checklist (`apps/mobile/store/play/en-GB/submission-checklist.md` section 4) says target audience 18+,
no children's audience; the App Store note (`app-store/en-GB/metadata.md`) expects a 12+ rating, which is a
content rating, not the audience. The policy says 18+.

## 2. Data inventory (from code)

Everything is in Supabase (Postgres, Ireland per the brief) unless noted. "Deleted" = removed when the account
is deleted, either by `PERSONAL_DATA_TABLES` or by `ON DELETE CASCADE` from `auth.users` (checked in the
migrations and the FK catalogue, below).

| Category | Tables / place | Why | Processor | Retention | Deletion |
|---|---|---|---|---|---|
| Account | `auth.users`, `user_profiles` (email, display_name, role, is_active, access_source) | sign-in, roles | Supabase | until deletion | explicit delete + `auth.admin.deleteUser` |
| Sessions | `auth.sessions` (ip, user_agent) | sign-in | Supabase | session life | cascade |
| Invitations | `invitation_uses`, `invitation_codes` | invite-only access | Supabase | until deletion | list + code retired (`retireReferralCodes`) |
| Progress snapshot | `learner_app_state` (state jsonb) | sync | Supabase | until deletion | list |
| Progress rows | `learner_lesson_progress`, `learner_unit_progress`, `learner_concept_state`, `learner_read_completions`, `learner_sessions`, `learner_events` | progress, mastery | Supabase | until deletion | list / cascade |
| Review cards | `review_cards` | revision | Supabase | until deletion | list |
| Answers | `assessment_attempts` (raw_submission, scores, rubric, AI score), `privacy_assessment_commits`, `viva_sessions`, `scan_results` (the last two have no route that writes them) | marking, history | Supabase; Anthropic sees the text | until deletion | list / cascade |
| Classes | `classes`, `class_members`, `institution_memberships`, `institution_invitations` | classes | Supabase | until deletion | cascade (`privacy_withhold_account` first revokes) |
| Class email invitations | `class_invitations` (email) | invite by email | Supabase | not purged on expiry | `privacy_forget_class_invitations`, cascade for inviter |
| Live | `live_sessions`, `live_attendance`, `live_questions`, `live_poll_answers` | lectures | Supabase (+ Realtime) | until deletion | cascade |
| Messages | `notifications`, `faculty_messages` (sender nulled; name set to a generic string), `notification_preferences` | inbox | Supabase | until deletion | list / cascade / tombstone |
| Push | `push_devices` (token, device_name), `push_outbox`, `push_deliveries` (token) | push | Supabase; Expo; Apple/Google | until deletion (inactive tokens stay) | `push_outbox` in list; devices/deliveries cascade |
| Devices | `account_devices` (device_key, label, platform, last_seen) | device limit | Supabase | until deletion (removed rows stay) | list |
| Installs | `app_installations` | version tracking | Supabase | until deletion | list |
| Screenshots | `screen_capture_events` | anti-sharing | Supabase | until deletion | list |
| AI usage | `model_runs` (user_id, hashes, cost); Redis budget key `medicalos:budget:<user>:<month>` | cost, caps | Supabase; Upstash | rows until deletion; key 40 days | `model_runs.user_id` SET NULL (kept, unlinked); Redis key NOT deleted |
| Rate limits | Upstash Redis (`medicalos:*` prefixes, `analytics: true`) keyed by user id or IP | abuse | Upstash | window length; analytics retention not set in code | not deleted |
| Logs | Railway (Fastify logger: IP, path, status; some audit lines carry user id) | ops | Railway | not in code | not deleted |
| Audit | `privacy_private.access_audit` (actor uuid; student uuid in `details`), `content_*` audit tables | accountability | Supabase | indefinite | content audit actors SET NULL; `access_audit` NOT cleaned (gap 2) |
| Lecturer content | `curriculum_*`, `faculty_profiles`, `content_credits`, `topic_feedback`, rights ledger | teaching | Supabase | withheld, ledger up to 90 days (`privacy_rights_due`) | withheld + tombstoned |
| AI figures | OpenAI image API, faculty-only | figures | OpenAI | not in code | n/a (no student data) |
| Device-local | phone AsyncStorage/files; web localStorage; Windows `auth/session.bin` (AES-256-GCM) + Credential Manager (data key, device key, installation id) | app function | none | sign-out deletes session; uninstall leaves Credential Manager entries | user action |

NOT collected (each checked): location (no location module or permission), advertising id (no ad SDK),
analytics or crash reporting (no Sentry, PostHog, Firebase Analytics, Amplitude, Mixpanel or similar in
`apps/*/package.json`, `Cargo.toml`, or source), camera/microphone/contacts/photos (no module;
`READ_MEDIA_IMAGES` is blocked in `app.config.ts`), cookies (no `document.cookie`; fonts are self-hosted, so no
Google Fonts request).

## 3. Deletion gaps and things that surprised me (legal issues; owner decisions)

1. **Supabase Auth's own event log keeps the email address after deletion.** On the local stack
   (`medicalos-contracts`), `auth.audit_log_entries` held `user_deleted` and `login` rows with the user's email
   in the payload and no link to the deleted user (36 such rows). `DELETE /account` does not touch it. Hosted
   Supabase behaviour and retention are NOT VERIFIED. Migration 094 (branch `claude/deletion-gaps`, not merged)
   adds a function that removes the deleted user's entries; section 4.3.
2. **`privacy_private.access_audit` keeps a deleted student's account id.** `actor` has no foreign key, and
   `class_join`, `class_request`, `default_institution_joined`, `class_invitation_admission` (088, 089, 091, 092)
   write the student's id; `class_approve`/`class_revoke` (043) put the student id in `details`. Nothing deletes
   these (the 062 clean-up only touches rows tied to a rights ledger). It is a random id with the account gone,
   but it is a retained identifier. Migration 094 (not merged) scrubs it; the pages are worded for that (section 4.3).
3. **No time-based deletion of student data anywhere.** `learner_events`, devices, installs, screenshot reports,
   attendance, answers are kept until the user deletes the account. `docs/DEVICE-LIMIT.md`'s 30/14-day figures
   only free a device slot; the row stays. Nothing deletes an inactive account. Recommendation in section 4.1.
4. **Switched-off push tokens are kept.** Unregister/sign-out only sets `active=false` (`routes/push.ts`).
   Tokens and the device name (often "Nihal's Pixel") stay until the account is deleted. `push_deliveries` copies
   the token as text.
5. **Class invitations are never purged.** An invitation holds a third party's email. Once expired (30 days) or
   revoked, it stays until the invitee deletes an account with that address, or the inviter deletes theirs. An
   address that never signs up is kept indefinitely.
6. **Upstash Redis is not touched by account deletion.** The AI budget key (user id + month) has a 40-day TTL;
   rate limiters use `analytics: true`, whose stored per-identifier usage has no TTL set in code. NOT VERIFIED
   what Upstash retains.
7. **`topic_placements.placed_by`** (076) is a staff user id with no foreign key; it survives deletion of that
   staff member. `account_devices.removed_by` (an admin id) likewise. Both staff-only, low risk.
8. **`PERSONAL_DATA_TABLES` is not the whole list.** `push_devices`, `class_members`, `classes`,
   `institution_memberships`, `live_*`, `privacy_assessment_commits`, `learner_read_completions`,
   `faculty_update_settings`, `atlas_*` and `class_invitations.invited_by` go only through `ON DELETE CASCADE`
   from `auth.users`. They do delete (FKs seen in migrations 021, 026, 042, 063, 079, 082, 091 and in the FK
   catalogue of the local stack), but a future table with a plain FK would be silently retained or would block
   the delete. Worth a test that reads the FK catalogue; not made (out of scope for a policy draft).
9. **Old policy claim "no AI transcripts" was true only of model prompts.** Typed answers are stored in
   `assessment_attempts.raw_submission` indefinitely (until deletion) and visible to the lecturer who teaches
   the topic.
10. **The M1 note says identifying push content is closed.** Checked: the only push text is generic
    ("Lecture started", "New faculty message"), and messages are signed "Your instructor", not by name. Policy
    says so. Source: migrations 048 and 093 override the sender-name push text that 039 first wrote.
11. **Windows `COMPUTERNAME` is sent as the device name** and shown in the device list and to admins. It often
    contains a person's name. Disclosed.
12. **Credential Manager entries survive uninstall** (device key, installation id, data key) for both builds, and
    the NSIS uninstaller does not remove them either (`T6-CONTRACT.md` row 10). Disclosed with removal steps.
13. **Device limit state.** `docs/DEVICE-LIMIT.md`: mode is `observe` (records, refuses nothing) by default. The
    policy and terms now say a student account "may be limited" to two devices, true in observe mode and when enforced (section 4.1).

## 4. Every `[OWNER: ...]` marker, settled (second pass, 2026-10-15)

Sources are cited as "per X's published terms as of 2026-10-15" where they are public documents. Infrastructure
facts were read-only: `railway status --json` (plan, region), `railway variables --json` (variable NAMES, and the
host part of three URLs, never a value), `supabase projects list` (project region), DNS lookups of public hostnames.

### 4.1 Resolved with evidence (the placeholder is gone from the HTML)

| Marker | Decision written into the page | Evidence |
|---|---|---|
| Railway region | Netherlands (EU West, Amsterdam) | `railway status --json`: service `@medicalos/api`, `multiRegionConfig` = `europe-west4-drams3a`, 1 replica. Railway's regions page (docs.railway.com/deployments/regions) maps that id to "EU West Metal, Amsterdam, Netherlands" |
| Netlify region | Global CDN | `netlify.toml`: static build (`apps/web/dist`) with one SPA redirect; no Netlify Functions or Edge Functions in the repo. Site host `medicalos.netlify.app` is `FRONTEND_URL` on Railway |
| Upstash region | Europe (London) | Host from `UPSTASH_REDIS_REST_URL` (host only: `allowing-primate-20037.upstash.io`) is a CNAME to `p2-global.upstash.io` -> `global-latency.upstash.io` -> `global-euw2.upstash.io`; its IPs reverse to `ec2-*.eu-west-2.compute.amazonaws.com` (London). Same answer from 8.8.8.8, 1.1.1.1, 9.9.9.9, but this PC is in the UK, so a latency-routed answer is expected here. **Confirm in the Upstash console** that the primary region is eu-west-2 and that no read replica is outside Europe |
| Supabase region | Ireland (EU West), kept | `supabase projects list`: project `medicalos` (ref `ccst...`, the same ref as the host in Railway's `SUPABASE_URL`) has `region: eu-west-1`. It was not a marker; now confirmed rather than assumed |
| Anthropic region + terms | "United States and other regions Anthropic operates in (we do not restrict the region)"; deleted within 30 days; not used for training by default | Code: `apps/api/src/services/model-registry.ts` calls `api.anthropic.com` with no `inference_geo`, so the default `global` applies (platform.claude.com/docs/en/manage-claude/data-residency). Retention 30 days and no training on commercial API data by default: privacy.claude.com articles 7996866 and 7996868. Flagged inputs can be kept up to 2 years; not stated in the policy (it is the exception, and we send no identity). **Not verified for our org:** that nobody opted the Console workspace into data sharing (default is off) |
| OpenAI region + terms | "United States and other regions OpenAI operates in"; up to 30 days abuse-monitoring retention; no training | developers.openai.com/api/docs/guides/your-data: abuse-monitoring logs "up to 30 days", API data "not used to train", image endpoints (`gpt-image-2`) 30 days; no regional constraint unless a project is created with data residency, which this one is not (no regional host in `image-generation.ts`). Faculty-only, no student data |
| Expo region | "Mainly the United States" | expo.dev/privacy: processing in the US for non-US users, DPF + SCCs; no stated retention for push tokens. docs.expo.dev FAQ: notification content is held only in memory and queues. The policy does not state an Expo retention period because Expo publishes none |
| Sign-in email provider | Owner confirmed (2026-10-15, via the coordinator): custom SMTP is OFF; sign-in emails go through Supabase's built-in email service. The separate "Email delivery" row is removed; the Supabase row now says it sends sign-in links. Supabase's docs (supabase.com/docs/guides/auth/auth-smtp) do not name the underlying mail service or its region, so the page says only "Supabase" | Owner statement. **When custom SMTP is set up on the owner's own domain, the policy must name that provider and its region in the processors table (and in the section 8 transfers table), and this page must be re-dated.** **Risk, not for the page:** the same Supabase doc says the default service is "not meant for production use", is limited to "2 messages per hour", has no SLA, and "will refuse to deliver messages to addresses that are not part of the project's team". That last restriction would stop sign-in emails reaching students who are not on the Supabase team. Check that email sign-in actually works for a non-team address; if it does not, custom SMTP is needed sooner |
| Railway log retention | 7 days | `railway status --json`: `"plan": "hobby"`. docs.railway.com/observability/logs: Trial 7, Hobby 7, Pro 30, Enterprise up to 90 days. A Railway staff forum reply says all logs are stored 90 days with the plan only limiting what you can query; the docs are the published figure and the policy uses them. Upgrading to Pro changes this to 30 |
| Supabase Auth log retention + deleted users | Event table: kept while the account exists, entries naming the account id or email deleted on deletion; platform logs "no longer than 28 days" | Table `auth.audit_log_entries`: docs list no automatic expiry (supabase.com/docs/guides/auth/audit-logs). Deletion: migration 094 `privacy_forget_auth_audit` (see 4.3). Platform log retention by plan, supabase.com/pricing as of 2026-10-15: Free 1 day, Pro 7, Team 28, Enterprise 90. The plan of org `cloudzzy` is not readable with the CLI, so the policy uses the Team ceiling. Tighten to 7 days if you are on Pro |
| Backup rotation | "within 30 days" | Supabase docs (supabase.com/docs/guides/platform/backups): daily backups Pro 7 days, Team 14, Enterprise up to 30; PITR add-on 7/14/28; Free has no managed daily backups. 30 is the ceiling of every paid plan. Tighten to "7 days" if on Pro |
| Lecturers and built-in answers | Yes, lecturers can see them, for students who are active members of one of their classes. Page now says so | `supabase/migrations/042_faculty_privacy.sql`: `privacy_can_teach` is true for any active user on a `shared_core` root; `privacy_can_review_attempt` additionally needs the actor to be faculty owning an unarchived class in which the learner is an approved active member. `053`: policy `privacy_attempt_teacher` (SELECT on `assessment_attempts` to `authenticated`) and `privacy_override_attempt` both use it. `install-shared-privacy.ts` installs the built-in lesson checks as `shared_core` templates, and `081`'s attempt insert carries `access_root_id`. `apps/api/src/db/client.ts`: `/faculty/attempts` runs as the caller, so RLS applies. Read from master `8985a61`; **not exercised against a database** |
| NSIS uninstaller | Removes program + shortcuts; "Delete the application data" box (off by default) deletes `%APPDATA%\org.medicalos.desktop` and `%LOCALAPPDATA%\org.medicalos.desktop`; never the Credential Manager entries | `apps/desktop/src-tauri/tauri.conf.json`: `targets: ["nsis"]`, `installMode: currentUser`, no custom template or hooks; Tauri 2.12.0 (`Cargo.lock`). Tauri's stock `installer.nsi` (github.com/tauri-apps/tauri, crates/tauri-bundler): `un.ConfirmShow` adds the unchecked box, then `RmDir /r "$APPDATA\${BUNDLEID}"` and `RmDir /r "$LOCALAPPDATA\${BUNDLEID}"`; the box is skipped in passive and silent mode (data kept). `T6-CONTRACT.md` row 10 and `secrets.rs` comment: no hook removes keyring entries. **Not run**: I did not run an NSIS uninstall (only the LOCAL build may be installed) and read the template from `dev`, not from the exact 2.12.0 tag |
| Device limit (terms) | Terms and policy now say a student account "may be limited" to two devices; no marker | `railway variables --json` names: neither `MOS_DEVICE_LIMIT_MODE` nor `MOS_MAX_DEVICES` is set on Railway production, so code defaults apply: `observe` (records, refuses nothing) and 2 (`apps/api/src/services/device-limit.ts:62-66`; `docs/DEVICE-LIMIT.md`). "May be limited" is true today and stays true when enforced, so the page does not need to wait. Production values themselves were not read |
| Legal entity + address | CLOUDZZY LTD, 320 Firecrest Court, Centre Park, Warrington, WA1 1RG, United Kingdom, used exactly as the owner gave it. On the privacy policy (section 1 and the contact block), the terms (section 1 and contact), the deletion page (section 2) and the support page (Get help) | Owner's message via the coordinator, 2026-10-15. The owner's D-U-N-S number and Partner Center organisation id are deliberately not on any page |
| Grievance Officer name/address | CLOUDZZY LTD acts as Grievance Officer; contact is the email plus the company address | Recommended, not legal advice. DPDPA rules need the business contact of a person who can answer questions, not a named officer. The older in-app `apps/web/src/components/legal/PrivacyPolicy.tsx` names `privacy@medicalos.in` as Grievance Officer; that component is not imported anywhere (dead code) and the mailbox may not exist. Not used |
| Lawful basis + UK GDPR (rewritten: the controller is a UK company) | Section 4.1 of the policy: UK GDPR and DPA 2018 apply to all processing wherever the user is; Article 6 basis per data category (contract for account, progress, answers, classes, messages and AI marking; consent for the push token; legitimate interests for devices, installs, screenshot reports, watermark, logs, limits and the content-change record; legal obligation for requests); no special category data; rights under Articles 15 to 22 listed; complaint to the ICO (ico.org.uk); DPDPA framing kept for India (consent paragraph, section 10, Board complaint) | Recommended, not legal advice. Keep a short legitimate-interests assessment on file for the three LI rows. AI marking is not an Art. 22 decision (a score and feedback, not a decision with legal or similar effect) |
| Transfers from the UK (new policy section 8 table) | Safeguard named per provider; UK -> Ireland/Netherlands is covered by UK adequacy regulations for the EEA | Supabase: SCCs + UK ICO addendum (supabase.com/privacy). Railway: DPA s.9.4 incorporates the UK Addendum; DPF only if certified (railway.com/legal/dpa). Netlify: DPF + UK Extension + SCCs (netlify.com/privacy s.5, s.13). Anthropic: DPA Schedule 3 Part B incorporates the UK Addendum (anthropic.com/legal/data-processing-addendum); it does not claim DPF. Expo: self-certified DPF + UK Extension (expo.dev changelog 2024-12-17, expo.dev/privacy). OpenAI: SCCs + UK Addendum per a secondary summary of its DPA (openai.com refused my fetch with 403). Upstash: DPA only; its privacy PDF links to dataprivacyframework.gov but I could not read its text. **Verify before publishing:** (a) each DPA is actually in force for CLOUDZZY LTD's account (Anthropic/Railway/Supabase/Netlify DPAs usually apply automatically with the terms; confirm in each console); (b) Expo's and Netlify's active status on dataprivacyframework.gov, UK Extension column; (c) OpenAI's DPA text; (d) Upstash's mechanism, and swap the row's wording if it is the DPF or SCCs |
| India-restricted countries | "As of the date of this policy, none has been notified" | Recommended, not legal advice. DPDPA s.16(1) is a negative list: transfers are allowed except to countries the Central Government notifies. Searched 2026-10-15: no notification found, and law-firm and compliance sources (Sept 2026) say s.16 and Rule 15 are not yet in force (18-month phase, expected 2027-05-13). I did not read the Gazette. Re-check on publication day |
| Inactivity period | Recommendation applied: no inactivity deletion now; the page says it will be reviewed when MedicalOS opens beyond invited students, and that users will be told in the app first | Recommended. The DPDP Rules' inactivity-erasure schedule covers e-commerce, gaming and social media above user thresholds, not an invite-only education app. If you prefer a number, replace the sentence in section 9 row 1 with "We delete an account after 24 months without sign-in, and tell you by email 30 days before"; that needs a nightly job that does not exist |
| Dates (8 places) | One token, `[OWNER: date]`, in the header and footer of the four pages | `sed -i 's/\[OWNER: date\]/15 October 2026/g' privacy-policy-medicalos.html delete-account-medicalos.html terms-medicalos.html support-medicalos.html` (use the real date). The markers are inside `<mark>`; delete the `<mark>` tags too: `sed -i 's#<mark>\(15 October 2026\)</mark>#\1#g'` |
| Delete page: audit log identifiers | Page says the audit-log entries stay but lose the account identifier | Depends on 094 (4.3) |
| Delete page: periods | One sentence: backups within 30 days, server logs 7 days, sign-in event entries deleted on deletion, platform logs within 28 days, Upstash counters within 40 days | The rows above; Upstash is 4.3 item 6 |

### 4.2 Findings made while settling these (not in the first pass)

1. **Rate-limit analytics in Upstash have no expiry.** `apps/api/src/middleware/redis.ts` creates 16 limiters with
   `analytics: true`. In the installed `@upstash/ratelimit` 2.0.8, `Analytics` passes `retention: "90d"` to
   `@upstash/core-analytics` 0.0.10, whose `ingest` is only `ZINCRBY <prefix>:events:<hour>` with a JSON member
   holding the identifier (user id or IP) and no `EXPIRE` anywhere in that package. So hourly zsets of user ids and
   IPs accumulate for ever, and account deletion does not touch them. Nothing in the repo reads them (no
   `getUsage`/`getAllowedBlocked`); only the Upstash console dashboard does. The first-pass inventory said
   "analytics retention not set in code" and could not say more. **Read from source; not looked at on the production
   Redis.** The policy's "rate-limit counters: seconds to a day" is true of the counters but not of these records
   until the follow-up in 4.3 item 6.
2. **The owner holds a copy of production user data.** `C:\Users\Nihal\dev\medicalos-backups\` has
   `prod-2026-10-08-pre-cutover.dump` (per the cutover memory: a full dump "with user data" for the rehearsal).
   It is outside every provider and outside account deletion, and it makes "backups rotate within 30 days" false for
   that file. Delete it (and the rehearsal stack `medicalos-rehearsal`, which is a prod copy) once no longer needed, or
   the 30-day sentence in section 9 is untrue.
3. Anthropic API inputs flagged by its safety systems can be kept up to 2 years (privacy.claude.com 7996866). We send
   no identity with them, so the page does not mention it.
4. Railway: the docs say Hobby logs are kept 7 days but a staff reply says 90 days stored. The page uses the docs.

### 4.3 Wording that depends on the deletion-fix PR (branch `claude/deletion-gaps`, migration 094)

State read 2026-10-15: `MedicalOS-deletion-gaps` has migration `094_data_retention.sql` and
`supabase/tests/094_data_retention.seed.sql` **untracked and uncommitted**; no API change yet (no tracked file
differs from `a6d0d0e`). The sentences below are true only once the migration is applied (by the owner) and the
API code that calls its functions is deployed. Do not publish the page before then, or take these sentences out.

| # | Page sentence | What 094 (or the PR) must do | Source |
|---|---|---|---|
| 1 | Class invitations: "An invitation that expires or is withdrawn is deleted 7 days later. One that is accepted is deleted 7 days after you accept it, once the app has told you that you have joined, and in any case within 30 days" | `privacy_purge_class_invitations` (grace 7 days; accepted: 7 days and `announced_at`, or 30 days), nightly 21:30 UTC via pg_cron | 094 section 1, 6 |
| 2 | Push tokens: "Deleted when you sign out. A token Apple or Google report as no longer valid is deleted about a week later" (corrected, see 4.5) | API deletes the row at unregister/revoke (stated in the 094 header, **not yet in code**); `privacy_purge_push_devices` deletes inactive rows older than 7 days | 094 header, section 2 |
| 3 | Audit log: "keeps its entries but loses your account identifier" (policy section 11, delete page section 4) | `privacy_forget_account_references` -> `scrub_access_audit(user)`: nulls `actor`, replaces the id inside `details` with the nil uuid; nightly sweep for earlier deletions. API call in `DELETE /account` **not yet in code** | 094 section 3 |
| 4 | Sign-in event log: "entries that name your account identifier or email address are deleted when you delete your account" (policy section 9, delete page section 4) | `privacy_forget_auth_audit(user, emails)` called after the auth user is deleted. 094 itself says: **NOT VERIFIED on hosted Supabase** that the role can delete from `auth.audit_log_entries`. Test it on the first real deletion | 094 section 4 |
| 5 | `topic_placements.placed_by` / `account_devices.removed_by` | Not on the page (staff-only); 094 adds `ON DELETE SET NULL` foreign keys | 094 section 5 |
| 6 | Rate-limit counters expire by themselves, "the longest within 40 days"; nothing identifying kept in Upstash after that | **Not in 094** ("NOT DONE HERE ... Upstash keys"). Needs: `analytics: false` on the 16 limiters in `middleware/redis.ts` (nothing reads the analytics) plus a one-off removal of existing `medicalos:*:events:*` zsets in Upstash | 4.2 item 1 |

Also by design, 094 does not purge `account_devices` rows of devices a user signed out; the page says they stay until
the account is deleted.

### 4.4 What is left for the owner

1. **ICO data protection fee.** A UK company that processes personal data on computer normally has to pay the ICO fee (and register) unless an exemption applies; the fee tier depends on size and turnover. Check at ico.org.uk/for-organisations/data-protection-fee and register CLOUDZZY LTD. Not for the published page.
   **Representatives.** No UK representative is needed for a UK controller. An EU representative under EU GDPR Art. 27 may be needed if EU users are targeted (offering the service to people in the EU); the apps are on worldwide stores but MedicalOS is invite-only and built for India. The page says so and names no representative. Counsel to decide. Not for the published page.
   **Terms of Service, section 11, say Indian law and Indian courts**, but the provider is now a UK company. I did not change it (a legal choice). Counsel to decide whether the Terms should name England and Wales for UK/overseas users and India for Indian users.
2. ~~Sign-in email provider~~ settled (custom SMTP off; Supabase sends). Revisit when custom SMTP is added (see 4.1).
3. **Publication date** (one token, `[OWNER: date]`, 8 places; command in 4.1).
4. **Inactivity choice:** the recommendation (none now, review at wider launch) is already on the page; say if you want a number.
5. **Plan tiers (optional tightening):** Supabase plan -> backups "within 30 days" becomes 7 (Pro) or 14 (Team), platform logs
   "28 days" becomes 7 (Pro). Railway is Hobby (read from the CLI); upgrading to Pro makes its logs 30 days.
6. **Confirm in consoles:** Upstash primary region/replicas; Anthropic Console workspace is not opted into data sharing.
7. **Counsel review recommended** of: the UK GDPR section and lawful-basis table, the transfers table, the "no restricted country" sentence, the DPDPA consent wording, the Grievance Officer arrangement, and whether lecturers/institutions are independent controllers of the class rosters and attendance they download (the page treats CLOUDZZY LTD as the controller of everything).
8. Merge/apply the deletion-fix PR (4.3) before publishing, and delete the local production dump (4.2 item 2).

### 4.5 Aligned with 094 (deletion-fix engineer's report, branch `claude/deletion-gaps`, commit 2726629c; read 2026-10-15)

Edits made to the draft after that report:

| # | Change on the pages | Why |
|---|---|---|
| 1 | Push tokens (policy section 9): "Deleted when you sign out; one that Apple or Google report as no longer valid is deleted about a week later". Added: `push_outbox` records of queued notifications stay until account deletion; `push_deliveries` go with the device row. Removed "or switch notifications off" | Only sign-out deletes a token at once. Switching off lecturer messages only changes a preference. The nightly job deletes inactive tokens a week old |
| 2 | Class invitations: added "the clean-up runs nightly, so each deletion happens within about a day of these dates" | The job runs once a day (21:30 UTC) |
| 3 | Sign-in provider event log (policy section 9, delete page section 4): "When you delete your account we remove, as far as we can, the entries that contain your account identifier or email address; anything missed is removed by a nightly clean-up, as far as we can." The earlier exclusion for dashboard deletions and the "stay until we remove them" fallback are dropped | PR #224 (migration 094 after review fixes): `DELETE /account` removes the entries matching the id and any address no living account holds, fail-soft; a fail-soft nightly sweep removes entries naming accounts that no longer exist, including accounts deleted from the dashboard, and matches unreadable payloads as text. "As far as we can" stays: hosted `postgres` DELETE on `auth.audit_log_entries` is verified only by the owner's check (step 4 below). Earlier clauses ("normal log retention", "stay until we remove them") were replaced as unverified or now unnecessary |
| 4 | Rate-limit counters sentence kept ("expire by themselves, the longest within 40 days") | Becomes true only after the owner runs `node scripts/purge-ratelimit-analytics.mjs` (dry run first, then `--apply`) and the deployed code has `analytics: false`. The per-user AI-budget key has a 40-day TTL and is not deleted at account deletion; the figure covers it |
| 5 | Delete page "what is kept, and why": new bullet on staff reviewer identifiers in content-rights records (`rights_ledger.former_owner`, `review_decisions.reviewer`, `object_withdrawal_manifests.reconciled_by`, `shared_asset_rights_reviews.reviewed_by` and `.revoked_by`). Staff only. Access-audit wording unchanged | Those columns keep a staff id by design (094 header: "four reviewer columns in privacy_private that keep a staff id by design") |
| 6 | Inactivity: unchanged ("no inactivity deletion now") | The engineer's options are recorded for the owner |

**PRE-PUBLISH owner actions (the pages are untrue until done), in the order PR #224 gives:**
1. Apply `094_data_retention.sql` OFF-PEAK (the owner applies migrations; it adds foreign keys to `auth.users`, a brief SHARE ROW EXCLUSIVE lock).
2. `supabase functions deploy push-dispatch --no-verify-jwt`.
3. Merge PR #224, which deploys the API.
4. Check the hosted behaviour: in the SQL editor `select has_table_privilege('postgres','auth.audit_log_entries','DELETE')`; delete a throwaway account; confirm Railway logs no `account deleted, with warnings` and that its log entries are gone. **Until this passes, "as far as we can" is the honest limit of the sign-in event-log sentence.**
5. Run `node scripts/purge-ratelimit-analytics.mjs` as a dry run, then with `--apply`, against production Upstash; re-run the dry run and expect zero keys (this makes the rate-limit sentence true).
6. Publish the updated pages (replace `[OWNER: date]`, command in 4.1).

Standing duty after publishing: if a deletion ever logs a `privacy_forget_auth_audit` warning and the nightly sweep does not clear it, remove that user's entries by hand.

## 5. What I did not verify

- No production access of any kind. The FK catalogue and the `auth.audit_log_entries` observation are from the
  local stack `medicalos-contracts` (read-only queries; it may differ from master by the other session's
  branch). Migration files were cross-checked for the FKs listed in gap 8.
- Hosted Supabase, Railway, Netlify, Upstash, Anthropic, OpenAI, Expo behaviour and regions.
- That lecturers cannot reach more student data than section 5 lists: I read the routes that touch
  `learner_*` and `assessment_attempts`; I did not audit every RLS policy.
- Play Data Safety and App Store privacy label contents (they must match this policy; the device-limit doc
  already notes both need updating for the device id, model and screenshot events, and for push tokens).
- The rendered pages in a browser (HTML was balance-checked and every anchor resolves; the stylesheet and
  classes are the existing ones). No BrowserOS check: these are static legal pages, not the app.
- Legal accuracy of the DPDPA wording (consent, legitimate uses, nominee, Board complaint) and of the UK GDPR section:
  drafted, not reviewed by counsel.
- Second pass: no production Supabase, Upstash, Anthropic or OpenAI console was opened. Provider facts are from the
  providers' published pages as of 2026-10-15 (cited in section 4) and from `railway status/variables --json`,
  `supabase projects list`, `supabase orgs list` and DNS; the Supabase plan and the production SMTP setting could not be read.
- The HTML was tag-balance checked with a small parser after the second pass; not opened in a browser.
