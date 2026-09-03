Building and Breaking an OIDC SSO Flow with Keycloak

I self-hosted Keycloak locally via Docker and built two OpenID Connect clients (app-one, app-two) inside a dedicated realm, then ran a full Authorization Code flow by hand — no SDK, just raw HTTP requests — to understand exactly what happens under the hood when an app "logs you in with SSO."

After authenticating once against app-one, I hit the authorization endpoint for app-two in the same browser session. No login prompt appeared — Keycloak issued a new authorization code instantly, and both requests shared the same session_state value. That's the mechanism behind single sign-on made visible: one authentication event at the identity provider, silently trusted by every app configured to accept it.

I exchanged the authorization code for tokens via curl and decoded the ID token to inspect its claims directly: iss (which realm issued it), aud (which client it's scoped to), sub (the user's permanent identifier, separate from their username), and the identity claims (name, email, preferred_username) that OpenID Connect adds on top of plain OAuth 2.0.

What broke: my first token exchange attempt failed with unauthorized_client — invalid client credentials. The client secret I was using belonged to a client accidentally created in Keycloak's master realm instead of my dedicated lab realm — a mismatch that wasn't obvious until I checked the realm switcher and found lab-realm had never actually been created on the first attempt.

Fix: recreated the realm explicitly, rebuilt both clients inside it, pulled the correct client secret from the right realm, and reran the exchange — this time successfully.

I also wrote and tested 5 least-privilege AWS IAM policies, including one verified live against a real S3 bucket: ListBucket/GetObject succeeded, DeleteObject correctly failed with AccessDenied since it was never granted.
