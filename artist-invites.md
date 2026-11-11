# Artist Invites (invite-only onboarding)

Artists join the platform **by invite only** in the first iteration. Customer
registration stays open (storefront visitors can still create accounts to
track commissions); only artist/studio creation is gated.

## The flag

`ARTIST_REGISTRATION_OPEN` (`.env.example`):

- unset / anything but `'true'` → **invite-only** (the default)
- `'true'` → the pre-existing self-serve `/onboarding` flow works as before

The flag is read by `isArtistRegistrationOpen()` in `src/lib/security/flags.ts`. It is
an _enforcement_ switch, not just UI: both the page and the server action
check it independently.

## Admin workflow

1. In the Payload admin (System & Security → **Invites**), click **Create
   New**. The create screen is a single card:
   - enter the artist's **email** — the invite only works with this exact
     address;
   - **Send invite** asks you to confirm the address, then emails the claim
     link immediately (an invite's email is immutable after creation, so a
     typo cannot be corrected later — only revoked and reissued);
   - **Generate link** mints the invite without emailing anything and shows
     the claim link with a **Copy link** button, so it can be pasted into a
     messenger, DM, or any other channel.

   Expiry defaults to **7 days** and is set server-side; the role is always
   `artist` and is never asked for.

   If that address already has a **pending, unexpired** invite, no second one
   is created: both buttons return the existing link (`reused: true`) so one
   artist never has two live tokens. Revoke the pending invite first if you
   really want a fresh one.

2. The created invite's edit view shows the same link and copy button, plus
   `expiresAt` (extend it to give the artist more time) and `status` (set
   `revoked` to kill the link before it is used).
3. The link is `https://<platform>/invite/<token>` — single-use, valid 7 days
   by default.
4. Statuses: `pending` → `accepted` (auto, on successful provisioning);
   `revoked` is the manual kill switch.

## Claim flow

1. Artist opens `/invite/<token>` (platform host only — studio hosts redirect).
2. The page validates the invite server-side (exists, `pending`, not expired,
   email not already registered). Invalid states render an explanatory card;
   a valid invite redirects to `/invite/<token>/claim`.
3. The claim Route Handler (`invite/[token]/claim/route.ts`) is the **only**
   writer of the `artist_invite` cookie: a Server Component may not mutate
   cookies, so the page cannot set it. The handler re-validates the invite,
   installs the httpOnly cookie (1 hour) and redirects to `/onboarding`. Any
   failure bounces back to the claim page, which owns the messages, and no
   cookie is installed.
4. `/onboarding` (when the flag is closed) requires that cookie, re-validates
   the invite, pre-fills the email (locked in the account step) and hands the
   token through the wizard.
5. `registerArtistWithAI` is the real boundary: when the flag is closed it
   requires `inviteToken`, validates it against the submitted email, and
   **consumes it inside the same transaction** as user/studio provisioning.
   A failed provisioning leaves the invite `pending` (retryable); a successful
   one can never be replayed.

## Security properties

- Token: 256-bit `crypto.randomBytes(32).toString('base64url')`, stored
  plaintext as a bearer credential, never shown publicly (admin-only field).
- Admin identity is never an argument. `createArtistInvite` is a `'use server'`
  export, so its arguments are attacker-controlled request data; it accepts
  only `{ email, sendEmail }` and derives the caller from the session
  (`payload.auth({ headers })`). The injectable core that does the write lives
  in `src/features/invites/lib/create-invite.ts`, which is **not** a server
  module, so tests can pass a user without exposing a seam on the wire.
- Minting is rate-limited per IP (`inviteCreateRateLimiter`, 20/h; fail-open
  when Upstash is unconfigured) because the action endpoint is public even
  though the UI behind it is admin-only.
- An invite cannot exist without a valid email. `email` is hidden on the
  create view, and a field `admin.condition` makes Payload skip that field's
  own validation — including `required` — so the collection's `beforeValidate`
  re-checks presence and shape for every create path (composer, REST, scripts).
- One live token per address: minting looks for a pending, unexpired invite on
  the same (normalised) email first and returns that link instead of creating a
  second. The lookup runs as the acting admin (`overrideAccess: false`).
- Single-use under race: consumption is a conditional bulk update
  (`WHERE token = X AND status = 'pending'`), so two simultaneous claims
  cannot both win. If the consume matches zero rows (revoked/used mid-flight),
  provisioning aborts and the transaction rolls back — the invite stays
  claimable by the invitee.
- Claim page **and** claim handler are rate-limited per IP (`apiRateLimiter`,
  shared with the other auth endpoints; fail-open when Upstash is
  unconfigured), because the handler is reachable directly, not only through
  the page.
- Expiry is enforced lazily at every validation point (`isInviteUsable`);
  no cron. The `expiresAt` date is also shown in the admin.
- No email enumeration: `/invite/<token>` only reveals state to someone who
  already holds the token.
- The cookie is httpOnly, sameSite=lax, secure in production, host-only — it
  is a claim ticket, never trusted without re-validation against the DB.

## What changes when the flag is closed

- `/onboarding` → redirects to `/platform-home` without a valid invite cookie.
- `registerArtistWithAI` → rejects without a valid, matching invite.
- CTAs hidden: platform nav "Create Studio", landing hero/CTA buttons, the
  studios page "Launch a New Studio" card (admins included — mint a self-invite
  or flip the flag if you need an extra studio), and the 404 "Claim Your
  Studio" button.
- Customer `/register` and `/platform/register` are unaffected.

## Verification

Automated: `tests/int/artist-invites.int.spec.ts` (invite usability, lookup,
atomic consume, URL building, flag parsing, the create core, the public
action's forged-argument rejection, and the collection's email guard) plus the
existing onboarding suites. `npx tsc --noEmit` clean.

Manual QA (needs a running dev server + DB):

1. With the flag unset: `/onboarding` redirects to `/platform-home`; nav has
   no "Create Studio"; `/platform/register` still renders.
2. Create an invite with **Generate link** → copy the link from the result
   panel → open it in a private window.
3. Claim: valid link → `/onboarding` with locked email; complete the wizard →
   user + artist + studio + theme created; invite is now `accepted`; the same
   link again → "not active".
4. **Duplicate guard:** invite the same address again while the first is still
   pending → the panel says the pending invite was reused and shows the *same*
   link; the list still has one row for that address.
5. **Send confirmation:** type an address, click **Send invite** → the card
   asks to confirm before anything is mailed; **Cancel** returns to the form;
   editing the address clears the confirmation.
6. Set `ARTIST_REGISTRATION_OPEN=true` → `/onboarding` works directly without
   a token (self-serve restored).

## Deploy notes

- Dev DB gets the `invites` table via `payload push` (already reflected in
  `src/payload-generated-schema.ts`). Production needs a migration:
  `yarn migrate:create` after this branch merges, then review the generated
  SQL before applying.
- The admin import map was regenerated (`InviteCreatePanel`,
  `InviteSaveButton`, `InviteLinkField`).

## Follow-ups (not built)

- Admin bulk CSV import for the first 10 invites (currently one-at-a-time).
- Auto-expiry job (lazy expiry is correct; a cron would only tidy the list).
- Customer-role invites (the `role` field is already shaped for it).
- Invite-specific metrics (issued → claimed → onboarded) on the admin
  dashboard for the go-live monitoring.
