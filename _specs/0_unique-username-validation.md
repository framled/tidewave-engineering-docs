# Spec for unique-username-validation

branch: feature/unique-username-validation

## Summary
- Guarantee that the `username` is unique across all Tidewave users.
- When a user saves their username, the system must check whether it is available. If someone else already uses it, nothing is saved, this message is shown, and three available alternatives are suggested:
  - English: "This username is taken"
  - Spanish: "Este nombre de usuario no está disponible"
- Usernames are case-insensitive and always stored in lowercase.
- Uniqueness is enforced by a dedicated Firestore **username registry** collection, where each document ID is a username. Claiming a username means creating its document, and Firestore refuses to create a document that already exists. This registry is the source of truth for uniqueness.
- The username rules are tightened as part of this work: 3–30 characters, using only letters (a–z), numbers (0–9), periods (.) and underscores (_), with extra rules for periods. Some names are reserved.
- There are no existing users yet, so no migration of existing duplicates is needed.
- Current state:
  - **tidewave-social**: the username is edited on two screens: complete profile (onboarding, `CompleteProfileScreen`) and edit profile (`EditProfileScreen`). Both use the `OrProfileForm` form. Today the only checks are that the field is not empty and that the format is valid (letters, numbers, hyphens and underscores). The profile is written directly to Firestore (`users/{uid}.profile.username`) without checking for duplicates.
  - **tidewave-backend**: at sign-up, the `createUserProfile` function assigns a default username from the local part of the email (or `user_<uid>` when there is no email), without checking for duplicates. The `updateUserProfile` function syncs Firestore changes to the Postgres `profiles` table. In the Prisma schema, `profiles.username` is optional and has **no** uniqueness constraint. Firestore security rules live in `apps/firebase/firestore.rules`.

## Functional Requirements

### Username registry
- A dedicated Firestore collection holds one document per claimed username. The document ID is the normalized (lowercase) username, and the document records only the owner's user ID.
- A username belongs to a user only if that user owns its registry entry. The username in the user's profile must always match a registry entry the user owns.
- Changing a username is one all-or-nothing operation: claim the new registry entry, update the profile, and release the old registry entry. If any step fails, nothing changes.
- This operation must require a connection. It must never be queued offline and reported as successful before the server confirms it.
- Postgres (`profiles`) mirrors the username from Firestore and must always hold the same value for each user. Postgres does not decide uniqueness.

### Registry protection (security rules)
- A user can only create a registry entry for themselves, and only for a username that follows the format rules.
- A user can only release (delete) their own registry entries.
- A user can only change their profile username to a name whose registry entry they own after the operation, and the old entry must be released in the same operation.
- Clients can look up a single registry entry (to check availability and validate suggestions) but cannot list the collection.
- Reserved entries cannot be created, changed or deleted by any user.

### Uniqueness
- The username must be unique across the whole platform.
- Uniqueness is case-insensitive. Usernames are normalized to lowercase before they are checked and saved, so "Juan" is saved as "juan" and blocks any other casing of the same name.
- Leading and trailing whitespace is removed before checking and saving.
- Two users cannot end up with the same username even if they save it at the same moment. Whoever loses gets the "taken" message.
- If the user saves their own username unchanged, it is not flagged as taken and the registry is not touched.

### Format rules
- Length: minimum 3, maximum 30 characters.
- Allowed characters: letters (a–z), numbers (0–9), periods (.) and underscores (_). Hyphens and spaces are not allowed anywhere.
- A username cannot start or end with a period.
- A username cannot contain more than one period in a row (for example, "nombre..algo" is not allowed).
- A username cannot both start and end with double underscores (for example, "__name__"), because Firestore does not allow that pattern as a document ID.
- The existing "invalid username" message is updated to describe the new rules, in English and Spanish.

### Reserved names
- Reserved names are stored as pre-claimed entries in the same registry, owned by the system instead of a user. Because they already exist, nobody can claim them, and the uniqueness check covers them without extra logic.
- The initial list includes: admin, tidewave, owner, moderator, support, contact. The engineering team maintains it through a backend seeding process, not from the app.
- Trying to use a reserved name shows the same "taken" message.

### Account lifecycle
- When an account is deleted, its username enters a **7-day grace period**. During that time the registry entry stays in place, marked as belonging to a deleted account, so nobody can claim it and impersonate the previous owner.
- After the 7 days, the registry entry is removed automatically and the username becomes available to other users.
- Names in the grace period behave like taken names: claiming one shows the "taken" message, and they are never offered as suggestions.
- When an account is suspended, its registry entry stays in place, so the username remains reserved for as long as the suspension lasts.

### Saving flow (complete profile and edit profile)
- The availability check happens only when the user taps save/continue. There is no live checking while typing.
- On save:
  - The format validations (required, length, allowed characters, period rules) run first.
  - Then the username is claimed in the registry together with the profile update.
  - If the username is taken or reserved, no profile changes are saved, "This username is taken" / "Este nombre de usuario no está disponible" is shown next to the username field with three suggested alternatives, and the user stays on the screen.
  - If the claim succeeds, the profile is saved and navigation continues as it does today.
- While the save runs, the save button shows a loading state and does not accept repeated submissions.
- The message is added to the existing translation system (English and Spanish) and shown in the user's active language.

### Suggested alternatives
- When a username is taken, the system shows **three** available alternatives based on what the user typed.
- First choice: add numbers to the end of the requested name (for example, user1, user2).
- If those are not available, fall back to adding random text or one of the sport names the app already has.
- Every suggestion must follow the format rules (including the 30-character maximum) and must not exist in the registry when shown.
- Tapping a suggestion fills the username field. It does not save. The user still taps save, which runs the normal claim.

### Default username at sign-up
- The username assigned automatically at sign-up must follow the same rules and must be claimed in the registry as part of creating the profile.
- Characters not allowed in the email local part (for example, hyphens or plus signs) are removed or replaced, leading, trailing and consecutive periods are cleaned up, and the result is shortened or padded to fit the length rules.
- If the resulting name is taken, the system claims an available alternative using the same suggestion strategy, without user intervention.

## Possible Edge cases
- Case differences ("Surfer" vs "surfer") resolve to the same registry entry.
- Leading or trailing whitespace ("juan " vs "juan").
- A suggestion that becomes taken between being shown and being saved: the claim fails, the "taken" message is shown, and three new suggestions are offered.
- Requested names near the 30-character limit: suggestions must shorten the base name so the suffix fits.
- Names that differ from a reserved name only by casing ("Admin", "ADMIN").
- Email local parts that become too short or invalid after cleanup (for example, "a-b@mail.com", "x@mail.com", ".juan@mail.com").
- Orphaned registry entries: a claim exists but no profile points to it (for example, a failed sign-up). Sign-up cleanup must release the claim, and orphans must not block names forever.
- A user claiming many usernames without using them (name squatting). The rules allow only the user's current username to be held.
- A deleted account's username being reused after the grace period: the new owner must not inherit anything (followers, mentions, posts) from the previous owner.
- A suspended account that is later deleted: the 7-day grace period starts at deletion.
- The automatic removal after 7 days fails or runs late: the name stays blocked a bit longer, but must never be released early.
- Lost connection during save: the operation fails, nothing is saved, and the existing generic error is shown, not the "taken" one.
- The user changes the username while a previous save is still running.
- Direct writes to Firestore that bypass the app (modified client): the security rules must reject a profile username that has no matching registry entry owned by the user.
- Users who abandon onboarding and resume it later (persisted onboarding state) with a username someone else took in the meantime.
- Postgres keeps soft-deleted profiles (`deleted_at`). A released username must be reusable there too, without conflicting with the deleted row.

## Acceptance Criteria
- Saving a username that belongs to another user shows "This username is taken" (English) or "Este nombre de usuario no está disponible" (Spanish), shows three suggested alternatives, and does not modify the profile or the registry.
- Tapping a suggestion fills the username field without saving.
- Saving an available username claims it in the registry, saves the profile, releases the previous username, and navigation continues as it does today.
- Saving the profile without changing one's own username works without errors and does not touch the registry.
- A username typed with uppercase letters is saved in lowercase. "Juan" is blocked if "juan" already exists.
- Usernames shorter than 3 or longer than 30 characters, with characters other than a–z, 0–9, "." and "_", starting or ending with a period, or with consecutive periods are rejected with the updated invalid-format message.
- Reserved names (in any casing) cannot be saved and show the "taken" message.
- Every suggested alternative is valid and available when shown.
- A deleted account's username shows the "taken" message for 7 days after deletion and can be claimed by another user after that. A suspended account's username cannot be claimed.
- Two new sign-ups with the same email local part get different, valid usernames, each with its own registry entry.
- Two users trying to save the same username at the same moment do not both end up with it.
- A client that writes a profile username directly, without owning the registry entry, is rejected by the security rules.
- After saving, the username matches in the registry, the Firestore profile and the Postgres `profiles` table.
- With no connection, saving fails with a generic error and can be retried. Nothing is queued.

## Open Questions
- None at this time.

## Testing Guidelines
Create test files following each repo's current test structure (backend functions: next to the source in `src/`; backend security rules: `apps/firebase/tests/`; backend core API: next to the source; app: co-located `__tests__/` folders), and create meaningful tests for the following cases, without going too heavy:

- Saving an available username claims it, saves the profile and releases the previous one.
- Saving a taken username shows the message in the active language, shows three suggestions, and changes nothing.
- Tapping a suggestion fills the field without saving.
- Saving one's own username unchanged does not flag it as taken.
- Uppercase input is saved as lowercase, and casing or whitespace variants map to the same registry entry.
- Format rules are enforced: length 3–30, allowed characters, no leading or trailing period, no consecutive periods, no spaces or hyphens.
- Reserved names are rejected in any casing.
- Suggestions follow the format rules and are available.
- Security rules: users cannot claim names for others, delete others' entries, touch reserved entries, list the registry, or set a profile username they don't own.
- A deleted account's username stays blocked during the 7-day grace period and is released after it. A suspended account's username stays reserved.
- Signing up two users with the same email local part produces different, valid, claimed usernames.
- Two simultaneous claims for the same username: only one succeeds.
- Saving while offline fails and is not queued.
- The sync to `profiles` keeps the same username as Firestore.
