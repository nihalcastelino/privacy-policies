# MedicalOS privacy update: draft changes (branch `privacy-update-2026-10`)

DRAFT for the owner's review. Not pushed, not published. The policy is the owner's legal document: every
`[OWNER: ...]` marker below is a fact the code cannot settle. Delete this file before merging to `main`
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

Kept as is: contact `nihal@cloudzzy.co.uk`, controller name "MedicalOS", DPDPA framing, 30-day response,
18+ / not directed at children, Apple Hide My Email paragraph, backups paragraph (with a rotation placeholder).

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
   Supabase behaviour and retention are NOT VERIFIED; check the dashboard. The policy has a placeholder.
2. **`privacy_private.access_audit` keeps a deleted student's account id.** `actor` has no foreign key, and
   `class_join`, `class_request`, `default_institution_joined`, `class_invitation_admission` (088, 089, 091, 092)
   write the student's id; `class_approve`/`class_revoke` (043) put the student id in `details`. Nothing deletes
   these (the 062 clean-up only touches rows tied to a rights ledger). It is a random id with the account gone,
   but it is a retained identifier. The delete page discloses it with a placeholder; a clean-up would be a
   migration (not made).
3. **No time-based deletion of student data anywhere.** `learner_events`, devices, installs, screenshot reports,
   attendance, answers are kept until the user deletes the account. `docs/DEVICE-LIMIT.md`'s 30/14-day figures
   only free a device slot; the row stays. Nothing deletes an inactive account (DPDPA expects erasure when the
   purpose is served). Placeholder in section 9.
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
    policy and terms say "currently two"; the terms carry an owner marker so you can hold that until enforcing.

## 4. Owner markers (all `[OWNER: ...]`)

Privacy policy:
1. Effective date (header) and last updated (footer): `date of publication`.
2. Legal name and registered address of the data fiduciary (section 1).
3. Name/postal address of the Grievance Officer, if published (section 1).
4. What the NSIS uninstaller removes (section 2.9).
5. Lawful-basis wording (counsel), and whether EEA/UK distribution adds GDPR wording (section 4).
6. Whether lecturers can see answers to built-in (non-faculty) questions (section 5).
7. Regions: Railway, Netlify, Upstash, Anthropic (plus its retention and no-training terms), OpenAI, Expo
   (section 7); email-delivery provider and region (section 7).
8. Counsel check on cross-border transfers and restricted countries (section 8).
9. Inactivity retention period, if any (section 9).
10. Railway log retention (section 9).
11. Supabase Auth event-log retention, and whether deleted users' entries are removed (section 9).
12. Backup rotation period (section 9).

Delete page: date x2; accept or fix the `access_audit` identifier retention; periods for backups/logs/auth log.
Terms: date x2; confirm the shared-login / device-limit paragraph and the number "two".
Support: date x2.

The Supabase region "Ireland (EU West)" is NOT a marker: it comes from the brief (pooler host `aws-1-eu-west-1`),
not from code. Please confirm it in the Supabase dashboard; Edge Functions and Storage are assumed to be in the
same project region.

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
- Legal accuracy of the DPDPA wording (consent, legitimate uses, nominee, Board complaint): drafted from the
  existing policy's framing; needs counsel.
