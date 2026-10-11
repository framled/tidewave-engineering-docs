# Plan: Unique username validation

Spec: `tidewave-engineering-docs/_specs/0_unique-username-validation.md`

## Context

Today nothing stops two users from having the same username:
- The app writes `users/{uid}.profile.username` straight to Firestore.
- At sign-up, the backend copies the email local part into the username as-is (`apps/firebase/functions/src/index.ts:57`).
- Firestore rules let the owner write any field.
- Postgres `profiles.username` has no constraint.

The spec makes usernames unique, lowercase and limited to 3–30 characters (`a–z 0–9 . _`, no leading or trailing period, no `..`).
- A Firestore **username registry** is the source of truth: `usernames/{name}`, where creating a name that already exists fails.
- Reserved names are pre-claimed.
- A deleted account's name is held for 7 days.
- A taken name shows "This username is taken" / "Este nombre de usuario no está disponible" with 3 tappable suggestions.

The work spans two repos, each on its own `feature/unique-username-validation` branch: **tidewave-backend** (rules, functions, core API, shared types) and **tidewave-social** (app).

## Registry data model

`usernames/{name}`: the doc ID is the normalized username.

| Field | Meaning |
|---|---|
| `uid` | Owner's uid, or `"system"` for reserved names |
| `createdAt` | Server timestamp |
| `reserved` | `true` only on reserved docs |
| `deletedAt`, `expiresAt` | Set only when the owner's account is deleted. `expiresAt = deletedAt + 7d`, and it drives a Firestore TTL policy |

**Username format.** One canonical rule is used in the app, the security rules and the functions:
- 3–30 characters
- matches `^[a-z0-9_]+(\.[a-z0-9_]+)*$`
- does not match `^__.*__$` (Firestore reserves that ID pattern)

The rule is written as RE2 with no lookarounds, so it works unchanged in security rules.

---

## Backend: tidewave-backend

### 1. Firestore security rules: `apps/firebase/firestore.rules`
- Add helper functions: `isValidUsername(name)` and `usernameOf(data)`, which safely reads `data.profile.username` and returns null when it's missing.
- **`match /usernames/{name}`:**
  - `get`: any authenticated user. `list`: denied.
  - `create`: requires all of the following:
    - `request.resource.data.uid == request.auth.uid`
    - the data has only the `uid` and `createdAt` keys
    - `isValidUsername(name)`
    - `getAfter(/users/$(uid)).data.profile.username == name`. This forces the claim and the profile update into the same transaction and blocks claiming extra names.
  - `delete`: requires `resource.data.uid == request.auth.uid`, no `reserved` field, no `deletedAt` field, and `getAfter(users/uid)` no longer pointing at `name`.
  - `update`: denied.
- **`match /users/{userId}`:** split `write` into `create`, `update` and `delete`, keeping the owner-only check.
  - On create and update, if `usernameOf` changed, require both:
    - `getAfter(/usernames/$(new)).data.uid == uid`
    - `!existsAfter(/usernames/$(old))`, when an old username exists
  - Writes that leave the username unchanged pass as they do today.
  - Deny client `delete` of the user doc (deletion goes through the core API).

### 2. Firestore TTL: `apps/firebase/firestore.indexes.json`
- Add a `fieldOverrides` entry for collection group `usernames`, field `expiresAt`, with `ttl: true`. It deploys through the existing `firebase deploy --only firestore` in `.github/workflows/firebase-deploy.yml`.
- TTL deletes run some time after expiry (usually within 24h), never before. That meets the spec's "never early" rule.
- If the firebase-tools version in CI rejects `ttl`, fall back to a one-time `gcloud firestore fields ttls update` documented in the README.

### 3. Username helpers: new `apps/firebase/functions/src/utils/username.ts`
Pure functions:
- `normalizeUsername` (trim, lowercase)
- `isValidUsername`
- `sanitizeFromEmail`: removes characters that aren't allowed, collapses or trims periods, truncates to 30, and pads short names (for example with digits) to reach 3
- `candidateUsernames(base)`: `base`, then `base1..baseN` (truncating the base so the suffix fits in 30), then a random-suffix fallback

A small `SPORT_CODES` list is copied from `packages/resources/sports.json` (for example `surf`, `kitesurf`) for the fallback. Shared-types can't be imported at runtime in functions (see the `tsconfig` paths note), so the list has to be duplicated.

### 4. Sign-up: `createUserProfile` in `apps/firebase/functions/src/index.ts:40-127`
- Build the base name with `sanitizeFromEmail(user.email)`, or `user_<uid6>` when there's no email.
- For each candidate, up to about 10 attempts to stay inside the blocking function's deadline:
  - Run a `db.batch()` with `create(usernames/{c}, {uid, createdAt})` and `set(users/{uid}, firestoreProfile)` using `username: c`, then commit.
  - On `ALREADY_EXISTS`, try the next candidate.
- Reserved and grace-period names already exist as docs, so they're skipped automatically.
- Extend the existing catch block so it also deletes `usernames/{claimed}`.
- Use the claimed name in the Postgres upsert.

### 5. Postgres sync: `updateUserProfile` in `index.ts:133-194`
- Postgres is **not** the source of truth. It is a copy that `updateUserProfile` updates only after the username transaction has committed in Firestore. The existing trigger already fires when `profile.username` changes, so the flow stays the same.
- Line 164: write `afterData.profile.username ?? null` instead of `"unknown"`, so deleted accounts get `NULL`.
- No Prisma migration and no unique constraint on `profiles.username`. Uniqueness is decided only by the Firestore registry.

### 6. Account deletion grace period: `deleteUserAccount` in `apps/core/services/users.service.ts:148-181`
- Before deleting the Auth user, read `users/{uid}.profile.username`.
- Then run one Admin batch that:
  - adds `profile.username: null` to the existing anonymizing `set`, so old posts no longer show a name someone else may later own
  - updates `usernames/{name}` with `deletedAt: serverTimestamp()` and `expiresAt: now + 7 days`
- `suspendUserAccount` needs no change, because the registry doc simply stays.

### 7. Reserved names seed: new `apps/firebase/scripts/seed-reserved-usernames.ts`
- The list (`admin, tidewave, owner, moderator, support, contact, …`) lives next to the script.
- The script is idempotent: for each missing doc it runs `create` with `{uid:'system', reserved:true, createdAt}`.
- It works against the emulator when `FIRESTORE_EMULATOR_HOST` is set, and against prod otherwise.
- **Why a new seed is needed:** the pipeline's existing seed step ("Seed Database" in `.github/workflows/prisma-deploy.yml`, which runs `pnpm --filter @tidewave/prisma run seed`, i.e. `packages/prisma/prisma/seed.js`) seeds only Postgres through Prisma. It can't create Firestore registry docs, and its job has no Google credentials.
- Wiring, following the same pattern:
  - an `apps/firebase/package.json` script (`seed:usernames`)
  - a "Seed Reserved Usernames" step in `.github/workflows/firebase-deploy.yml` after the deploy step. That job already authenticates through `google-github-actions/auth` and picks dev/prod the same way `prisma-deploy.yml` does.
  - the script and reserved list added to `firebase-deploy.yml`'s `paths` trigger, so editing the list redeploys
  - a call from `scripts/wipe-data.js --reseed` for local emulators

### 8. Shared types: `packages/shared-types/src/models/profile.ts`
- Change `FirestoreUserProfile.profile.username` to `string | null`.
- Add `UsernameRegistryEntry`.
- Export it from `src/index.ts`, following the GEMINI.md rule to always update shared types.

---

## App: tidewave-social

### 9. Pure username utils: new `src/utils/username.ts`
- Same rule set as the backend: `normalizeUsername`, `validateUsername`, which returns `'required' | 'length' | 'chars' | 'period' | null`, and `candidateUsernames(base, sportCodes)`.
- Keep the backend and app copies in sync with a shared list of test inputs: the same valid and invalid examples in both test files.

### 10. Service: new `src/services/usernameService.ts`
Follows the `onboardingService.ts` style and uses `getFirestore()` from `./firebase`.
- **`saveProfileWithUsername(uid, username, docData)`** runs `runTransaction`:
  1. `tx.get(users/uid)` → `oldName`
  2. If `oldName === username`, just `tx.set(users/uid, docData, {merge:true})`.
  3. Otherwise, `tx.get(usernames/username)`. If it exists, throw `UsernameTakenError`.
  4. `tx.set(usernames/username, {uid, createdAt: serverTimestamp()})`
  5. `tx.set(users/uid, docData, {merge:true})`
  6. If there's an old name, `tx.delete(usernames/oldName)`.

  Transactions fail offline instead of queueing, which is what the spec requires. Contention retries cover two users saving the same name at once.
- **`suggestUsernames(base, sportCodes)`** walks the candidates and checks each one with `getDocFromServer(usernames/c)`, skipping cache. It stops after 3 available names, with a cap of about 15 lookups.

### 11. UI
- **New `src/components/molecules/MlUsernameSuggestions.tsx`:** a row of `AtChip` with `uppercase={false}` and `onPress`, labelled with `t('profile.usernameSuggestions')`. Both screens need it and nothing existing fits, so it gets the justification comment AGENTS.md requires.
- **`src/components/organisms/OrProfileForm.tsx`:**
  - Replace the hardcoded regex and English messages (L94-101) with `validateUsername` + `t()` keys.
  - Lowercase the username as the user types.
  - Add props `usernameError?`, `usernameSuggestions?` and `onSuggestionPress?`.
  - Make the username value respond to a parent change, so a tapped suggestion fills the field. The current `initialData` sync effect (L36-47) ignores username.
- **`src/screens/profile/CompleteProfileScreen.tsx` (L70-206):**
  - Replace the regex and `OrAlert` username checks with `validateUsername`.
  - Replace `setDoc` at L201 with `saveProfileWithUsername(userId, normalized, payload)`.
  - On `UsernameTakenError`:
    - set an inline error
    - load suggestions, using `route.params.selectedSports` codes for the fallback
    - stay on the screen
  - Other errors keep the existing `OrAlert` plus `getReadableAuthError`.
- **`src/screens/profile/EditProfileScreen.tsx` (L240-283):** the same flow. It builds its own `MlFormField` for the username (L369-375), so pass `error` there and render `MlUsernameSuggestions` below it. Use the loaded `sports` / `specialties` for the fallback. The `AtButton isLoading/disabled` behaviour already blocks double submits.

### 12. Translations: `src/i18n/translations.ts`
Edit both `en` and `es`:
- `profile.usernameTaken`: "This username is taken" / "Este nombre de usuario no está disponible"
- `profile.usernameInvalid`: reworded for the new rules
- `profile.usernameLength`, `profile.usernamePeriod`
- `profile.usernameSuggestions`

### 13. Test infrastructure: `jest-setup.js`
Add `runTransaction` and `getDocFromServer` to the global `@react-native-firebase/firestore` mock.

---

## Tests

Tests follow each repo's current structure:

| Repo / package | Location | Runner |
|---|---|---|
| backend functions | next to the source in `apps/firebase/functions/src/**/*.test.ts` | Jest, `testMatch: **/src/**/*.test.ts` |
| backend rules | `apps/firebase/tests/verify_rules.ts` | `@firebase/rules-unit-testing` against the emulator |
| backend core | next to the source, e.g. `apps/core/services/*.test.ts` | `deno test` |
| app | co-located `__tests__/` folders | Jest, `jest-expo` |

### Backend
- **`apps/firebase/functions/src/utils/username.test.ts`:**
  - normalize, validate and sanitize-from-email, including `a-b@`, `x@`, `.juan@` and `juan+1@`
  - candidates stay at 30 characters or fewer
- **New `apps/firebase/functions/src/createUserProfile.test.ts`:**
  - Mock the pattern from `processUploadedMedia.test.ts`.
  - The first candidate collides (`ALREADY_EXISTS`) and the second is claimed.
  - On failure, cleanup deletes both docs.
- **`apps/firebase/tests/verify_rules.ts`:** add cases:
  - claim own valid name
  - can't claim for someone else
  - invalid names: `.a`, `a.`, `a..b`, `AB`, `ab`, 31 characters, `__x__`
  - can't list
  - can't delete reserved, grace-period or others' docs
  - profile username change rejected without a registry doc
  - change with claim and release accepted
  - non-username profile writes still allowed

  Add a `test:rules` script: `firebase emulators:exec --only firestore "jest tests"` with a small jest config. It runs from `apps/firebase`; nothing runs this file today.
- **New `apps/core/services/users.service.test.ts`** (same style as `posts.service.test.ts`): a `deleteUserAccount` test that stubs Firestore and asserts that `expiresAt` is about 7 days out and `profile.username` is null.

### App (co-located `__tests__/`)
- **`src/utils/__tests__/username.test.ts`:** the same examples as the backend.
- **`src/services/__tests__/usernameService.test.ts`:**
  - available → claim + profile write + old name released
  - taken → `UsernameTakenError` and no writes
  - unchanged name → registry untouched
  - suggestions return 3 available names and skip taken ones
- **`src/screens/profile/__tests__/CompleteProfileScreen.test.tsx`:**
  - Update the existing test. It asserts on `setDoc`; it should assert the payload passed to `saveProfileWithUsername` instead.
  - Add the taken case: the error is shown, suggestions render, tapping one fills the field without saving.
- **`src/components/molecules/__tests__/MlUsernameSuggestions.test.tsx`:** renders chips and calls `onPress` with the right value.

## Verification
1. Backend:
   - `pnpm --filter @tidewave/functions test`
   - `pnpm --filter <core> test`
   - `pnpm --filter <firebase> test:rules`, which needs the Firestore emulator
2. App: `pnpm test` in tidewave-social.
3. End-to-end on the emulators:
   1. Start the backend through `compose.yml` and run `seed:usernames`.
   2. Run the app against the emulator (`src/services/firebase.ts` emulator setup).
   3. Sign up `juan@a.com` and `juan@b.com`. They should get `juan` and `juan1`, each with a `usernames/` doc.
   4. In Edit Profile, try:
      - `Juan`, which should show taken plus 3 suggestions
      - tapping a suggestion, which should fill the field
      - saving, after which the old doc is gone, the new doc exists, and Postgres `profiles.username` matches
      - `admin`, which should show taken
      - `.a`, `a.`, `a..b`, which should show the inline format error
   5. Turn on airplane mode and save. It should show the generic error, with nothing queued.
   6. Delete an account through `DELETE /me`. The registry doc should have `expiresAt` 7 days out, the name should show taken, and Postgres `username` should be NULL.

## Rollout order
1. Deploy the backend first: rules, indexes/TTL, functions, then run the seed.
2. Ship the app after that.

There are no existing users. Once the rules are deployed, an old app build can still save profiles if the username is unchanged; changing the username will be rejected.
