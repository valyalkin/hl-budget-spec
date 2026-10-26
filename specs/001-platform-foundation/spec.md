# Feature Specification: Platform Foundation

**Feature Branch**: `001-platform-foundation`

**Created**: 2026-10-25

**Status**: Implemented 2026-10-26 (all 89 tasks; deployed to the homelab at `hl-budget.home.example`). Deferred: prod Auth0 tenant, trusted TLS certificate, home/VPN DNS (T078)

**Input**: User description: "Platform foundation for HL Budget, a personal budget tracking web app that runs on my homelab. This first feature delivers the working skeleton only: a frontend, a backend and sign-in, deployed and healthy. It contains no budgeting features. Goal: prove that a user can sign in, reach a protected area, and see data that the backend returns for them specifically. Every later feature builds on this." (Full description: 7 user stories, 3 operator stories, one versioned profile endpoint, out-of-scope list, open questions and success criteria.)

## Clarifications

### Session 2026-10-25

- Q: Who may sign in? → A: Anyone who can sign in through Auth0, with no app allow-list. The app
  is private-network only and the operator is the only user for now. Revisit before going public.
- Q: Which sign-in methods? → A: Email/password and Google, self sign-up disabled (accounts
  created in advance).
- Q: Hostname layout? → A: One hostname, web app at `/` and server under `/api`. It is reachable
  only from the home network or the operator's VPN. The exact hostname is set at deploy time.
- Q: Session lifetime? → A (defaulted): 7 days of inactivity, 30 days absolute, no "stay signed
  in" option.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sign in and see my own profile (Priority: P1)

A visitor opens HL Budget and sees a public landing page with the app name, a one-line
description and a "Sign in" button. They sign in on the identity provider's hosted login page
and land on the home page (a dashboard placeholder). The home page greets them by name and shows
their email and avatar. These details come from HL Budget's own server, which recognises who they
are, not only from the login session. The first time this happens, HL Budget creates a user
record for them. Later visits update that record and never create a second one.

**Why this priority**: This is the goal of the feature. It proves sign-in, a protected area, and
user-specific data from the server all work together. Every later feature depends on this user
record.

**Independent Test**: Start signed out, click "Sign in", complete login, and check that the home
page shows the correct name, email and avatar. Sign in again as the same person and confirm there
is still exactly one user record, with its last-seen time updated.

**Acceptance Scenarios**:

1. **Given** a visitor who is not signed in, **When** they open the app's root address, **Then**
   they see the landing page with the app name, a one-line description and a "Sign in" button,
   and no user data.
2. **Given** a visitor on the landing page, **When** they click "Sign in" and complete login,
   **Then** they arrive at the home page.
3. **Given** a signed-in user on the home page, **When** the page loads, **Then** it shows a
   greeting with their name, their email and their avatar, as returned by the server.
4. **Given** a person who has never used HL Budget, **When** they first reach the server signed
   in, **Then** a user record is created for them, linked to their identity-provider account,
   with name, email, first-seen time and last-seen time.
5. **Given** a person who already has a user record, **When** they sign in again, **Then** their
   last-seen time is updated and no second record is created.
6. **Given** a user without an avatar at the identity provider, **When** the home page loads,
   **Then** a neutral placeholder (for example their initials) is shown instead of a broken
   image.

---

### User Story 2 - Protected pages require sign-in (Priority: P1)

Someone who is not signed in tries to open a protected page directly, for example from a
bookmark. They are sent to sign in and, once signed in, return to the page they asked for. The
server refuses to return user data to any request without valid credentials.

**Why this priority**: The app holds personal financial data. A protected area that can be
reached without signing in is a defect, however well everything else works.

**Independent Test**: While signed out, open a protected address and confirm you are sent to sign
in and then back to that address. Call the profile service with no credentials and with invalid
credentials, and confirm both are refused as unauthenticated.

**Acceptance Scenarios**:

1. **Given** a visitor who is not signed in, **When** they open a protected page, **Then** they
   are sent to sign in and see none of that page's content.
2. **Given** that visitor completes sign-in, **When** login succeeds, **Then** they land on the
   protected page they originally asked for, not the default home page.
3. **Given** a request to the server's profile service with missing, malformed, expired or
   wrongly issued credentials, **When** it is processed, **Then** it is refused as
   unauthenticated and no user data is returned or created.
4. **Given** a return address after sign-in that points outside HL Budget, **When** sign-in
   completes, **Then** the user is sent to the home page instead (no open redirect).

---

### User Story 3 - Sign out (Priority: P2)

A signed-in user chooses "Sign out". Their session ends in both HL Budget and the identity
provider, and they return to the landing page. Afterwards, opening a protected page requires
signing in again, and the identity provider asks for credentials again rather than silently
signing them back in.

**Why this priority**: Needed for a shared or borrowed device and for trust, but the core proof
(P1) works without it.

**Independent Test**: Sign in, sign out, confirm you are on the landing page, then open a
protected page and confirm you are asked to sign in and must enter credentials again.

**Acceptance Scenarios**:

1. **Given** a signed-in user, **When** they choose "Sign out", **Then** they arrive at the
   landing page and no longer appear signed in.
2. **Given** a user who has just signed out, **When** they open a protected page, **Then** they
   are sent to sign in.
3. **Given** a user who has just signed out, **When** they start sign-in again, **Then** the
   identity provider asks for their credentials instead of signing them in silently.
4. **Given** a user who has just signed out, **When** they use the browser's back button,
   **Then** no protected content that would need a session is shown.

---

### User Story 4 - Failed sign-in, expired sessions and errors are handled gracefully (Priority: P2)

Things go wrong in predictable ways. A user cancels or fails sign-in, their session expires, the
server rejects their credentials, the server is down, or they open an address that does not
exist. In each case they see a clear, friendly message or are sent to sign in. They never see a
crash, a raw error or broken data.

**Why this priority**: These paths do not add capability, but without them the skeleton looks
broken the first time something fails. Later features reuse the same patterns.

**Independent Test**: Cancel the login page and check the landing-page message. Let a session
expire, or make the server reject credentials, and check that you are sent to sign in. Stop the
server and load the home page to check the "something went wrong" state. Open a made-up address
to check the not-found page.

**Acceptance Scenarios**:

1. **Given** a visitor on the hosted login page, **When** they cancel or sign-in fails, **Then**
   they return to the landing page with a clear message saying sign-in did not complete and that
   they can try again.
2. **Given** a user whose session has expired, **When** they open or reload a protected page,
   **Then** they are sent to sign in and afterwards return to that page.
3. **Given** a signed-in user whose credentials the server rejects, **When** the home page loads,
   **Then** they are sent to sign in instead of seeing an error or partial data.
4. **Given** a signed-in user while the server is unavailable or returns an unexpected error,
   **When** the home page loads, **Then** it shows a clear "something went wrong" message with a
   "Try again" action, and no technical details are displayed.
5. **Given** that "something went wrong" state, **When** the server is available again and the
   user chooses "Try again", **Then** their profile loads normally.
6. **Given** anyone, signed in or not, **When** they open an address that does not exist,
   **Then** they see a not-found page with a link back to the start.

---

### User Story 5 - Operator sees both services healthy after a deploy (Priority: P2)

The operator, who runs the homelab, releases a new version. Both services (the user-facing web
app and the server) roll out through the normal release pipeline with no manual steps. They become
reachable at the app's hostname over HTTPS and report healthy. Each service reports whether it is
alive and whether it is ready to serve. The server reports not ready while its database cannot be
reached, and ready again once it can.

**Why this priority**: The constitution requires every service to meet the homelab contract from
day one, and "deployed and healthy" is part of this feature's definition of done. It ranks below
the user journey because it can be checked against the same build.

**Independent Test**: Release a version through the pipeline, and with no manual intervention
confirm both services show healthy in the platform and the app loads over HTTPS at its hostname.
Make the database unreachable and confirm the server reports not ready while still reporting
alive. Restore the database and confirm it reports ready again.

**Acceptance Scenarios**:

1. **Given** a new release, **When** it goes through the normal release pipeline, **Then** both
   services are deployed and report healthy with no manual steps.
2. **Given** deployed services, **When** someone opens the app's hostname over HTTPS, **Then** the
   landing page loads with a valid certificate. Plain HTTP is redirected to HTTPS or refused.
3. **Given** either service is running, **When** its liveness check is called, **Then** it reports
   alive without checking dependencies.
4. **Given** the server's database is unreachable, **When** the server's readiness check is
   called, **Then** it reports not ready, and reports ready again within one check interval after
   the database comes back.
5. **Given** a running service, **When** it is asked to stop, **Then** it finishes in-flight
   requests and exits cleanly within 30 seconds.

---

### User Story 6 - Developer runs everything locally (Priority: P3)

A developer clones the project and, following documented steps, runs the web app, the server and
a database on their own machine. Example configuration lists every setting the services need,
with placeholder values and no real secrets.

**Why this priority**: Important for working on later features, but it delivers no value to end
users and can be finished after the deployed path works.

**Independent Test**: On a clean machine with the documented prerequisites, follow the
documented steps with your own development identity-provider settings, and confirm you can sign
in locally and see your profile.

**Acceptance Scenarios**:

1. **Given** a developer with the documented prerequisites, **When** they follow the local setup
   steps, **Then** the web app, server and database all start and the sign-in journey works
   end to end locally.
2. **Given** the repository, **When** anyone inspects the example configuration and committed
   files, **Then** every required setting is listed and described, and none holds a real secret.
3. **Given** a required setting is missing, **When** a service starts, **Then** it fails fast with
   a message that names the missing setting, not a vague error.

---

### Edge Cases

- **Same person, two browsers or two tabs at once**: two first-time requests arriving together
  must still produce exactly one user record.
- **Name or email changes at the identity provider**: on the next visit the stored name and email
  are updated to match. The record stays linked to the same identity, never to an email address.
- **Missing profile fields**: the identity provider returns no name or no avatar. The greeting
  falls back to the email address, or to a neutral greeting, and the avatar falls back to a
  placeholder.
- **Same email, different sign-in method**: one email reached through two different
  identity-provider accounts (for example a password account and a Google account). These are
  treated as separate identities unless the identity provider links them. See Assumptions.
- **Session expires mid-visit**: the next navigation or data load sends the user to sign in,
  never to a broken page.
- **Server slow but up**: the home page shows a loading state and, after a reasonable timeout,
  the "something went wrong" state rather than waiting forever.
- **Signed-in user opens the landing page**: they are offered a way into the app (or sent to the
  home page) rather than being asked to sign in again.
- **Database down while users are signed in**: the home page shows the "something went wrong"
  state, and the server reports not ready.
- **Someone tries to create an account**: self sign-up is disabled (FR-020), so the login page
  offers no sign-up path. A person without a pre-created account cannot get past the login page,
  and no user record is created for them.
- **Opened from outside the private network without VPN**: the app is not reachable at all. This
  is expected behaviour, not an error the app handles.

## Requirements *(mandatory)*

### Functional Requirements

**Public and protected areas**

- **FR-001**: The system MUST show a public landing page, without sign-in, containing the app
  name ("HL Budget"), a one-line description and a "Sign in" action.
- **FR-002**: Every page other than the landing page, the not-found page and sign-in/sign-out
  handling MUST be protected. Opening one without a valid session MUST send the visitor to sign
  in and MUST NOT reveal any of its content.
- **FR-003**: After a successful sign-in, the system MUST return the user to the protected page
  they originally asked for, or to the home page if there was none. Return addresses MUST be
  limited to HL Budget's own pages.
- **FR-004**: Protected pages MUST be excluded from search-engine indexing.

**Sign-in, session and sign-out**

- **FR-005**: Users MUST sign in through the identity provider's hosted login page (Auth0). HL
  Budget MUST NOT collect or store passwords itself.
- **FR-006**: If sign-in is cancelled or fails, the system MUST return the user to the landing
  page with a clear, non-technical message saying sign-in did not complete.
- **FR-007**: The user's session MUST be kept so that their credentials for the server are never
  readable by scripts running in the browser.
- **FR-008**: A session MUST stay valid while the user is active and expire after 7 days of
  inactivity. Regardless of activity, it MUST expire 30 days after sign-in. No "stay signed in"
  option is offered in this feature.
- **FR-009**: A "Sign out" action MUST be available on every protected page. It MUST end the
  session in both HL Budget and the identity provider and return the user to the landing page.
- **FR-010**: When a session has expired, or the server rejects the user's credentials, the
  system MUST send the user to sign in (keeping the page they were on as the return address)
  instead of showing an error or partial data.

**Profile and user record**

- **FR-011**: The home page MUST show a greeting with the user's name, their email and their
  avatar, using data returned by the server for the authenticated user. Placeholders are shown
  where the name or avatar is missing.
- **FR-012**: The server MUST offer one versioned, authenticated profile operation that returns
  the current user's profile (identifier, name, email, avatar, first-seen and last-seen times).
  On the user's first call it creates their record.
- **FR-013**: On each call to the profile operation, the server MUST update the user's last-seen
  time and refresh the stored name, email and avatar from the verified identity.
- **FR-014**: The server MUST guarantee at most one user record per identity-provider identity,
  including when several first-time requests arrive at the same moment.
- **FR-015**: The server MUST identify the user only from verified credentials. It MUST check the
  issuer, intended audience, signature and expiry, and MUST NOT accept a user identity supplied in
  any other way.
- **FR-016**: Any request to the profile operation without valid credentials MUST be refused as
  unauthenticated (401). It MUST NOT return or create user data.
- **FR-017**: A user MUST only ever be able to see their own profile. The profile operation takes
  no user identifier as input.
- **FR-018**: The server's operations MUST be described in a published, machine-readable API
  document (OpenAPI). It covers every operation, its responses and its error responses, with
  examples. Errors MUST use one consistent structured format that includes a trace reference and
  exposes no internal details.

**Access control**

- **FR-019**: The system MUST accept any person who successfully signs in through the identity
  provider. There is no app-side allow-list in this feature. Who can sign in is controlled by the
  identity provider (accounts created in advance, see FR-020) and by the app being reachable only
  from the private network (see FR-024). This MUST be revisited before the app is exposed
  publicly.
- **FR-020**: The hosted login page MUST offer email/password and Google sign-in. Self sign-up
  MUST be disabled: accounts are created in advance in the identity provider, and the login page
  offers no way to create a new account.

**Errors and not-found**

- **FR-021**: If the server is unavailable, slow beyond a timeout, or returns an unexpected error
  while the home page loads, the page MUST show a friendly "something went wrong" message with a
  "Try again" action and no technical details.
- **FR-022**: Any unknown address MUST show a not-found page with a link back to the start.

**Operations**

- **FR-023**: Each service MUST expose a liveness check that does not depend on other systems, and
  a readiness check. The server's readiness check MUST report not ready while its database is
  unreachable.
- **FR-024**: Both services MUST be deployable to the homelab through the normal release pipeline
  with no manual steps. They MUST share one hostname over HTTPS, with the web app at `/` and the
  server under `/api`. The hostname MUST be reachable only from the home network or through the
  operator's VPN, not from the public internet. The exact hostname is deployment configuration,
  not fixed in code.
- **FR-025**: Services MUST take all configuration from their environment, fail fast with a clear
  message when a required setting is missing, write logs to standard output, and stop cleanly
  within 30 seconds when asked.
- **FR-026**: No secret MAY be committed to the repositories, built into images or written to
  logs. Each service MUST ship an example configuration file that documents every setting with
  placeholder values only.
- **FR-027**: The project MUST include documented steps for running the web app, the server and a
  database locally.

**Scope**

- **FR-028**: This feature MUST NOT include any budgeting capability (budgets, accounts,
  categories, transactions, imports, reports), settings pages, roles or admin features, email
  notifications, mobile apps or multi-currency support.

### Key Entities

- **User**: a person who has signed in to HL Budget at least once. It is linked one-to-one to an
  identity-provider identity, and that link is the user's permanent key (not their email). It
  holds an opaque app identifier, display name, email address, avatar link, first-seen time and
  last-seen time. Every future budgeting record will belong to exactly one User.
- **Session**: a signed-in visit in the web app. It ties a browser to a User's verified identity,
  has an inactivity limit and an absolute lifetime, and ends on sign-out. It is not stored by the
  server.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new user can go from the landing page to seeing their own profile on the home page
  in under 30 seconds, including time spent on the login page.
- **SC-002**: Signing in repeatedly as the same person, including from two browsers at once,
  produces exactly one user record in 100% of attempts.
- **SC-003**: 0 protected pages and 0 server operations return user data without a valid session.
  This is verified by automated tests covering signed-out, expired and invalid-credential cases
  for every protected page and operation.
- **SC-004**: After a release through the normal pipeline, both services report healthy and the
  app loads over HTTPS at its hostname from the home network and over VPN within 10 minutes,
  with 0 manual steps. The app is not reachable from the public internet.
- **SC-005**: For a signed-in user, the home page with their profile appears in under 2 seconds
  on the home network.
- **SC-006**: In each failure scenario tested (cancelled sign-in, expired session, rejected
  credentials, server down, unknown address), users see a friendly message or a sign-in prompt,
  and never a raw error page, a stack trace or a blank screen.
- **SC-007**: The server reports not ready within 30 seconds of losing its database, and ready
  again within 30 seconds of it returning.
- **SC-008**: A developer following the local setup documentation can sign in to a locally
  running copy in under 30 minutes on first attempt, with 0 real secrets found in the
  repositories.

## Assumptions

- **Identity provider**: Auth0 is the identity provider, as required by the project constitution.
  An Auth0 tenant and application exist, or will be created, for production and for local
  development.
- **Session lifetime** (open question, defaulted): 7 days of inactivity, 30 days absolute, with no
  "stay signed in" option. This is a common default for personal apps with low-sensitivity
  sessions and is easy to change later.
- **Access model** (clarified 2026-10-25): for now the operator is the only user, the app is
  private-network only (home network or VPN), and accounts are created in advance in Auth0.
  Revisit an allow-list, self sign-up and public exposure before the app goes public.
- **Account linking**: the same email reached through different sign-in methods is treated as
  different identities unless they are linked in the identity provider. HL Budget does not merge
  accounts by email.
- **Profile freshness**: name, email and avatar in the user record are refreshed from the verified
  identity on every profile call, so the identity provider stays the source of truth for these
  fields.
- **Personal scale**: a handful of users on a single homelab cluster. No high-availability,
  multi-region or heavy-load targets apply.
- **Platform**: the homelab already provides the release pipeline, internal DNS, HTTPS ingress
  with certificates that browsers on the private network and VPN trust, secret storage and a managed database. This feature consumes them and does not
  build them.
- **Browsers**: current versions of mainstream desktop and mobile browsers. The pages are
  responsive, but native mobile apps are out of scope.
- **Language**: the interface is English only.
- **Deletion**: deleting a user record or exporting data is out of scope for this feature.
