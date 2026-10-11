# Spec for orphan-signup-data

branch: fix/orphan-signup-data

## Summary
- Stop sign-up from leaving data behind for accounts that were never created, and clean up the orphan data that already exists.
- **Problem:** the `createUserProfile` function in tidewave-backend runs as a Firebase Auth *blocking* function (`beforeUserCreated`). It runs **before** Firebase Auth saves the account, and it already has side effects:
  - downloads the social profile photo to Storage (`users/{uid}/avatar.jpg`);
  - creates `users/{uid}` and claims `usernames/{name}` in Firestore;
  - inserts the row in the Postgres `profiles` table.

  If Auth then doesn't create the account, nothing undoes those writes. The function only cleans up when it fails itself.
- **How it showed up:** on 2026-10-11 at 23:30 UTC, tapping "Continue with Google" several times started several sign-ups for the same Google account. Auth kept one account (`fn.narvaezmichea`), and the other attempts left orphan data in `tidewave-dev`:
  - 6 uids with no Auth account;
  - 3 orphan `users/{uid}` profiles (`fn.narvaezmichea4`, `fn.narvaezmichea5`, `fn.narvaezmichea_canoeing`);
  - 6 claimed usernames that will never be released (`fn.narvaezmichea1`–`5`, `fn.narvaezmichea_canoeing`);
  - probably the matching Postgres `profiles` rows and Storage avatars (not yet verified).
- **Already fixed:** the app no longer allows repeated taps (tidewave-social PR #263). That doesn't cover network retries, two devices, or any other reason Auth might not create the account, so the backend fix is still needed.
- **Scope:**
  1. Create the profile only once the account exists.
  2. A one-time cleanup of the existing orphans.
  3. A safety net that finds future orphans.

## Functional Requirements

### Profile creation after the account exists
- The profile, username claim, Postgres row and social avatar are created only for accounts that Firebase Auth has actually created.
- The blocking sign-up function must have no side effects. Remove it, or keep it only for checks that can reject a sign-up without writing anything.
- Creating the profile must be idempotent. Running it more than once for the same account (event retries are at-least-once) must not:
  - claim a second username;
  - create a second Postgres row;
  - overwrite changes already made by the user or the app.
- Creating the profile must not overwrite fields the app has already written to `users/{uid}`. For example, the app saves `onboardingState` right after sign-in, which may happen before the profile is created.
- The default username rules from the unique-username spec still apply: taken from the email, cleaned up, claimed in the registry, with an alternative chosen automatically if the name is taken.
- The app must handle the short window after sign-in when the profile doesn't exist yet. The complete-profile screen must show the default username once it's available, and must not fail or save an empty profile in the meantime.

### One-time cleanup of existing orphans
- Find every uid that has data but no Firebase Auth account, across:
  - Firestore `users/{uid}`;
  - Firestore `usernames/{name}` entries owned by that uid;
  - Postgres `profiles`;
  - Storage `users/{uid}/`.
- **Deleted accounts are not orphans.** Accounts deleted through the app have no Auth account by design, but they keep an anonymized profile (`metadata.deletedAt`) and a held username (`deletedAt` / `expiresAt` in the registry). These must be left untouched.
- **Reserved names are not orphans** (`uid: "system"`, `reserved: true`).
- The cleanup first runs as a dry run that lists what it would delete. It only deletes after someone reviews that list.
- It runs on `tidewave-dev` first. Whether to run it on production is decided after reviewing the dry run there.

### Safety net for future orphans
- A recurring check finds uids that have data but no Auth account and are older than a safe margin (for example 1 hour), so sign-ups still in progress aren't affected.
- Whether it reports, cleans up, or both is an open question.

## Possible Edge cases
- Several sign-up attempts for the same Google or Facebook account at once (the original bug). Only the created account gets a profile.
- The profile-creation event is delivered twice: the second run is a no-op.
- The app writes `onboardingState` to `users/{uid}` before the profile is created: both end up in the document.
- The user opens complete-profile before the profile exists: no crash, and the default username appears once it's created.
- The user deletes their account before the profile-creation event runs: no profile or username is recreated for the deleted account.
- Email/password sign-up (no display name or photo) and social sign-up (with photo) both work.
- The social photo download fails: the profile is still created, without an avatar, as today.
- The default username's alternatives are all taken: the existing fallbacks apply, and if they also run out, the failure is logged and retried rather than leaving a profile without a username.
- An account linked to a second provider (same email, Google then Facebook) does not get a second profile.
- Orphan cleanup meets a soft-deleted account in its 7-day hold: it is skipped.
- Orphan cleanup meets a username in the registry whose uid has an Auth account but whose profile points to a different name, for example a leftover from a failed change. It is reported, not deleted, because it isn't an orphan by this definition.
- The Postgres user used by the functions may only have INSERT/UPDATE on `profiles` in production, so the cleanup may need different credentials to delete rows.

## Acceptance Criteria
- Repeating the original scenario (several simultaneous Google sign-ups for one account, for example by calling the API directly) produces exactly one Auth account, one `users/{uid}`, one claimed username, one Postgres row and at most one avatar.
- No Firestore, Postgres or Storage writes happen for a sign-up attempt that Auth does not complete.
- Delivering the profile-creation event twice for the same account leaves the same result as delivering it once.
- `onboardingState` written by the app right after sign-in is still present after the profile is created.
- The complete-profile screen works when opened before and after the profile exists.
- The cleanup dry run on `tidewave-dev` lists the 6 known orphan uids and their data, and does not list the deleted account (`vSZzGQotLedcEC6sBLhNDulARsD2`), its held username, or reserved names.
- After the cleanup, `fn.narvaezmichea1`–`5` and `fn.narvaezmichea_canoeing` are available again, and the orphan profiles no longer show up in the app (for example in user search).
- The safety net finds an orphan older than the margin and ignores sign-ups in progress.

## Open Questions
- Is a short delay between sign-in and the profile existing acceptable? The app may need to wait for the profile document to appear before showing complete-profile.
- Should the safety net only report orphans (logs or alerts), or also delete them automatically?
- Should orphan Storage avatars be part of the cleanup, or left for a separate Storage clean-up?
- Should the cleanup also run on production after reviewing its dry run?
- Do we keep the blocking function for any checks that should reject a sign-up (for example blocked email domains), or remove it entirely?

## Testing Guidelines
Create test files following each repo's current test structure (backend functions: next to the source in `src/`; backend security rules: `apps/firebase/tests/`; backend core API: next to the source; app: co-located `__tests__/` folders), and create meaningful tests for the following cases, without going too heavy:

- Profile creation runs only for a created account, and the blocking function (if kept) writes nothing.
- Running profile creation twice for the same account claims one username and writes one Postgres row.
- Profile creation keeps fields the app wrote earlier (`onboardingState`).
- The default username is claimed, with an alternative when taken.
- Cleanup classifies data correctly: orphan vs. deleted account in its hold vs. reserved name vs. active account.
- The cleanup dry run deletes nothing.
- The safety net ignores data newer than the margin.
- The complete-profile screen handles a profile that doesn't exist yet.
