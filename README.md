# IAM Lab — Keycloak OIDC SSO + AWS Least-Privilege Policies

Hands-on identity and access management lab covering OpenID Connect / SSO fundamentals and AWS IAM least-privilege policy design. Built as part of a self-directed transition from GRC into IAM, targeting Microsoft Entra ID (SC-300) and platform-agnostic IAM fundamentals.

## What's in this repo

- `policies/` — 5 real AWS IAM JSON policies, written and tested against a live AWS account
- `screenshots/` — proof of the SSO flow and policy enforcement (see below)

---

## Part 1 — Building and Breaking an OIDC SSO Flow with Keycloak

I self-hosted Keycloak locally via Docker and built two OpenID Connect clients (`app-one`, `app-two`) inside a dedicated realm, then ran a full Authorization Code flow by hand — no SDK, just raw HTTP requests — to understand exactly what happens under the hood when an app "logs you in with SSO."

After authenticating once against `app-one`, I hit the authorization endpoint for `app-two` in the same browser session. No login prompt appeared — Keycloak issued a new authorization code instantly, and both requests shared the same `session_state` value. That's the mechanism behind single sign-on made visible: one authentication event at the identity provider, silently trusted by every app configured to accept it.

I exchanged the authorization code for tokens via `curl` and decoded the ID token to inspect its claims directly:

| Claim | What it means |
|---|---|
| `iss` | Which realm/identity provider issued the token |
| `aud` | Which client the token is scoped to |
| `sub` | The user's permanent identifier — separate from username |
| `exp` / `iat` | Token expiry and issue time |
| `name`, `email`, `preferred_username` | Identity claims — the layer OpenID Connect adds on top of plain OAuth 2.0 |

**What broke:** my first token exchange attempt failed with `unauthorized_client — invalid client credentials`. The client secret I was using belonged to a client accidentally created in Keycloak's `master` realm instead of my dedicated lab realm — a mismatch that wasn't obvious until I checked the realm switcher and found `lab-realm` had never actually been created on the first attempt. The same mix-up resurfaced later since Keycloak lets you create two clients with the identical ID (`app-one`) in different realms, which cost more debugging time until I confirmed which realm's client secret actually matched the login URL I was using.

**Fix:** recreated the realm explicitly, rebuilt both clients inside it, pulled the correct client secret from the correct realm, and reran the exchange — this time successfully.

### Screenshots
- `screenshots/id-token-decode.png` — decoded ID token via jwt.io, showing real claims (`iss`, `aud`, `sub`, `name`, `email`)
- `screenshots/sso-app-one.png` — address bar after logging into `app-one`, showing `session_state`
- `screenshots/sso-app-two.png` — address bar after logging into `app-two` **without re-entering credentials** — same `session_state` value as above, proving SSO

---

## Part 2 — Least-Privilege AWS IAM Policies

Five policies covering distinct least-privilege patterns, written and tested against a live AWS account (not just theoretical JSON):

| File | Pattern demonstrated |
|---|---|
| `s3-readonly.json` | Resource-scoped read access — `GetObject` + `ListBucket` only, locked to one bucket ARN |
| `ec2-startstop.json` | Action-restricted operational access — start/stop only, no create/terminate, region-locked via `Condition` |
| `deny-iam-changes.json` | Explicit deny boundary — blocks IAM/org/account changes regardless of other permissive policies attached |
| `s3-delete-require-mfa.json` | Condition-gated permission — delete is granted but only enforced when `aws:MultiFactorAuthPresent` is true |
| `crossaccount-trust-policy.json` | Trust policy (not a permission policy) — defines *who can assume a role*, distinct from *what the role can do* |

### Verified live

Attached `s3-readonly.json` to a real IAM user (`test-readonly-user`) and tested both directions:

**Allowed (as written):**
```
aws s3 ls s3://wael-iam-lab-2026 --profile readonly-test
→ 2026-09-03 10:25:06  28  test.txt
```

**Denied (correctly blocked):**
```
aws s3 rm s3://wael-iam-lab-2026/test.txt --profile readonly-test
→ AccessDenied: User ... is not authorized to perform: s3:DeleteObject
  because no identity-based policy allows the s3:DeleteObject action
```

### Screenshot
- `screenshots/access-denied.png` — the AccessDenied error above, confirming the policy blocks exactly what it doesn't grant

---

## Key takeaways

- OAuth 2.0 handles *authorization* (what an app can do on a user's behalf); OpenID Connect adds *authentication* (who the user is) via the ID token
- SSO works because the identity provider trusts an existing browser session — not because credentials are re-sent to every app
- A trust policy (who can assume a role) and a permission policy (what the role can do) are two different mechanisms that are easy to conflate as a beginner
- Least privilege isn't just writing a narrow policy — it's verifying both the allow and deny paths actually behave as written
- Realm/client scoping in Keycloak (and directory/tenant scoping generally, e.g. Entra ID) is a common source of "invalid credentials" errors that has nothing to do with the credentials themselves — always confirm which realm or directory a client/secret actually belongs to before assuming it's broken
