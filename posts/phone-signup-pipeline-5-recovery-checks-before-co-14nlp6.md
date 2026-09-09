# Phone Signup Pipeline: 5 Recovery Checks Before Code Delivery and Verification

Short answer: gate code delivery with captcha, keep verification and session creation as separate state transitions, and design account recovery before accepting the first phone signup. For an education app, that is the least complex pipeline that blocks cheap bot registrations without making a lost phone equivalent to a lost course account.

| Decision | Safer default | Choose the alternative when |
| --- | --- | --- |
| Bot gate | Captcha before code delivery | An existing risk service already proves the signup action is human |
| Challenge | One active, expiring code per intent | Carrier delay data justifies a longer window under the same attempt cap |
| Account | Pending until verification | Legal or billing review must precede activation |
| Session | Fresh identifier after activation | A trusted identity provider owns session creation |
| Recovery | Enroll a durable alternate factor | The account is deliberately temporary and holds no durable progress |

The recommendation sits in the boundaries, not the vendor choice. Persist the signup intent and each allowed transition. Don't let a controller jump from a captcha result to an authenticated session because one six-digit string matched.

Small state machine. Sharp edges.

## What should a phone signup pipeline verify before session creation?

The pipeline should establish four different claims: this request may send a code, this challenge belongs to the intended signup, the submitted code is valid under current policy, and the verified account is eligible for a session. A single `POST /signup` handler can hide those distinctions. Once hidden, retries become guesswork and operators can't tell a delayed message from an exhausted challenge.

For an edtech signup, the captcha is an abuse gate rather than identity proof. Bind its successful result to one opaque signup intent and one action. Apply throttles across more than the phone number: a school network may put many legitimate learners behind one address, while a bot can rotate numbers. The exact combination depends on traffic you can measure; I'm not sure a fixed global threshold is defensible without that data.

Delivery needs an idempotency rule from the application's point of view. A double tap or client retry must not leave two concurrently valid challenges. On resend, advance the challenge version and invalidate the earlier value. Return a generic message such as "If this number can sign up, a code will arrive" so the endpoint doesn't become a convenient account-enumeration check. OWASP recommends generic authentication responses and defenses against automated attacks; registration deserves the same discipline.

The verification code is a secret with a narrow job. Store a keyed digest rather than plaintext, compare without leaking timing information, cap attempts, expire it, and record the consume transition. A six-digit code has only 1,000,000 possible values, so throttling and the attempt budget carry much of the security load. Code delivery is not verification. Verification is not activation. And activation still isn't session creation.

That last boundary matters during recovery. A recycled number, lost SIM, or changed guardian contact can separate a real learner from the factor used at signup. If recovery only sends another SMS, possession of the reassigned number becomes enough to take the account. If recovery is impossible, the learner can lose coursework. Enroll a durable alternate factor while the original session is trusted, keep guardian recovery data distinct from learner contact data, and revoke existing sessions before binding a replacement number. The specific proof may be a passkey, an authenticator, a verified email, or an institution review; what matters is that it doesn't collapse back to the unavailable phone.

## Two criteria that beat a long feature checklist

First, benchmark transition integrity. Can a resend invalidate the prior challenge atomically? Can two verification requests race and both succeed? Can a session exist before the account reaches the active state? Those three tests say more than a large matrix of authentication features because they probe the invariants the application actually owns.

Second, inspect the recovery dependency graph. A phone-first design with SMS-only recovery has one channel wearing two labels. That's config-light, but it is not a second path. A credible recovery route uses evidence independent of the unavailable number and leaves an audit event that support can understand without reading raw application logs.

I care about time-to-first-call, but I benchmark the glue after that call too. Count the persisted states, credentials, callback handlers, retry policies, and dashboards needed to answer one support question: why didn't this learner get access? Fewer moving parts help only while the key transitions remain observable. Once a wrapper erases provider outcomes or couples code verification to cookie issuance, the saved setup turns into operational debt.

## A TypeScript boundary that survives retries

Keep captcha verification and message delivery behind narrow interfaces. The state transition itself should be deterministic enough to test without either dependency.

```ts
type SignupState = "captcha_ok" | "code_sent" | "verified" | "active";

type Signup = {
  id: string;
  phoneDigest: string;
  state: SignupState;
  challengeDigest?: string;
  challengeExpiresAt?: number;
  attempts: number;
  version: number;
};

type VerifyResult =
  | { ok: true; signup: Signup }
  | { ok: false; code: "challenge_closed" | "code_mismatch" };

function verifyCode(
  signup: Signup,
  submittedDigest: string,
  now: number,
  maxAttempts = 5,
): VerifyResult {
  const closed =
    signup.state !== "code_sent" ||
    signup.challengeExpiresAt === undefined ||
    now >= signup.challengeExpiresAt ||
    signup.attempts >= maxAttempts;

  if (closed) return { ok: false, code: "challenge_closed" };

  const attempted: Signup = {
    ...signup,
    attempts: signup.attempts + 1,
  };

  if (submittedDigest !== signup.challengeDigest) {
    return { ok: false, code: "code_mismatch" };
  }

  return {
    ok: true,
    signup: {
      ...attempted,
      state: "verified",
      challengeDigest: undefined,
      challengeExpiresAt: undefined,
      version: signup.version + 1,
    },
  };
}
```

The equality check above operates on already-derived values to keep the example focused; production code should use an appropriate keyed construction and a constant-time comparison. Persistence must condition the update on `version`. If two requests read version 7, only one may write version 8 and consume the challenge. The loser should receive a stable domain result, not trigger another transition. That rule also makes retries boring, which is exactly what I want from auth code.

Session creation belongs in a separate transaction after the verified transition. Rotate the identifier, use `HttpOnly` and `Secure` cookies, choose `SameSite` according to the actual navigation flow, and never place a phone number, captcha token, or verification code in a URL. Log an opaque intent ID, tenant, transition, and reason code. Don't log the code or full phone number.

No magic here.

## When is the simpler recovery path actually better?

The catch is operational cost. Independent recovery factors create enrollment UI, support policy, audit retention, and cases where a person must review evidence. This design is not suitable for an anonymous practice tool whose accounts hold no purchase, certificate, private class history, or durable progress. In that case, stick with a temporary guest profile and request verified identity only when the learner crosses a boundary worth recovering.

A managed identity service may also be the better owner when an institution already controls authentication and recovery. Don't rebuild its account lifecycle inside the course app. Accept the trusted assertion, create an application session at that boundary, and keep bot controls on the public enrollment routes you still own.

For a durable consumer learning account, though, captcha plus SMS-only recovery is too circular. Add the alternate factor early. It is less painful during signup than during a locked-out student's deadline.

## Test the failures the happy path hides

Use table-driven tests for expired challenges, five failed attempts, captcha replay, resend races, duplicated delivery callbacks, phone normalization, and session fixation. Add a state-machine property: `active` must be unreachable unless a challenge was consumed, and no transition may move backward. A fake delivery adapter should be able to accept a request, delay it, and duplicate its callback without changing those invariants.

Metrics need reason codes rather than one `signup_failed` counter. Track `captcha_rejected`, `delivery_accepted`, `code_expired`, `code_mismatch`, `verification_consumed`, and `recovery_started` as transitions or outcomes. Alert on rates and changes over time, not one learner's failure. Keep the phone pseudonymous in telemetry, define retention with the education customer's requirements, and make an operator prove they can trace an intent without exposing the credential.

The decision rule is compact: choose the smallest phone signup pipeline that preserves delivery, verification, activation, and session boundaries while giving a real learner an independent recovery route. Add checks only in response to measured abuse or a concrete account-risk requirement.

## Further reading

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
