---
name: ziqx-auth-sdk
description: Integrate Sign in with Ziqx using @ziqx/auth in JavaScript or TypeScript apps. Use for S256 PKCE redirects, server-side authorization-code exchange, and JWT verification for Ziqx Developers apps and older clients.
---

# Ziqx Auth SDK

Install `@ziqx/auth`. Obtain the app key, client secret, and registered redirect URL from **Ziqx Developers → app → Auth**. The app key (for example `zdev_…`) is public; the client secret stays on the server. These credentials are separate from Drive and OTP keys.

## Choose the API

| API | Where it runs | Behavior |
| --- | --- | --- |
| `createPkce()` | Server or browser with Web Crypto | Returns `{ verifier, challenge, method: "S256" }`. |
| `new ZAuthClient({ authKey, baseUrl? })` | Server or browser | Configures the public app key and sign-in host. |
| `auth.getLoginUrl(options)` | Server or browser | Builds a URL without navigating. |
| `auth.login(options)` | Browser | Navigates with `window.location.href`. |
| `new ZAuthTokenService({ baseUrl? })` | Server or browser | Configures token exchange and verification. |
| `tokenService.getAuthToken(params)` | Server | Exchanges a code using the client secret. |
| `tokenService.verify(token, { appKey })` | Server or browser | Verifies ES256 signature, issuer, expiry, and audience; returns claims or throws. |

Use `getLoginUrl()` for a server redirect. Only `login()` requires a browser; the entire `ZAuthClient` class is not browser-only.

## Sign-in flow

1. Generate a fresh PKCE pair and unpredictable state for each sign-in attempt. Store the verifier and state in a short-lived server session or HttpOnly cookies until the callback.
2. Pass the **challenge**, `codeChallengeMethod: "S256"`, and state to `getLoginUrl()` or `login()`. The SDK does not store PKCE state or validate the callback for you.
3. On the callback, require a nonempty code, verifier, returned state, and stored state. Compare the returned and stored states before exchanging the code. Missing values must fail; two absent states are not a match.
4. Exchange the code on the server, using the original verifier and exactly the same redirect URL. Check which response variant was returned, then clear the temporary verifier/state and store the access token in your app's session.
5. Verify the session token on the server before granting access, passing your **app key** as the expected audience.

For a complete Next.js App Router implementation, read [references/nextjs.md](references/nextjs.md). It includes the login route, callback, and session helper; use it when working in Next.js rather than rebuilding a plain-PKCE or client-token pattern.

`LoginOptions` accepts:

- `redirectUrl`: the exact URL registered under Auth → Redirect URLs.
- `codeChallenge`: the challenge from `createPkce()`.
- `codeChallengeMethod`: use `"S256"`. Omitting it selects the legacy plain-verifier behavior.
- `state`: the value you will check against the stored state on callback.
- `scope`: a space-delimited string or array containing `openid` and optionally `email` and/or `profile`. The default is `openid email profile`; include `openid` when specifying scopes.

`getAuthToken()` takes `{ authAppKey, authSecret, code, codeVerifier, redirectUri }`. It resolves with `{ access_token, token_type: "Bearer", id_token }` or `{ error, ... }`. Use `"access_token" in result` to distinguish them; validation and network errors can still throw. Do not return the token to a client component simply to set a JavaScript-readable cookie.

## Verify a session

```ts
import { ZAuthTokenService } from "@ziqx/auth";

const tokenService = new ZAuthTokenService();

export async function getUser(token: string) {
  try {
    return await tokenService.verify(token, {
      appKey: process.env.ZAUTH_APP_KEY!,
    });
  } catch {
    return null;
  }
}
```

`verify()` accepts a leading `Bearer ` prefix. Tokens use the app **key** for `aud` and `azp`; the app ID is not the audience. The deprecated `appId` option only works when its value is actually the app key. The SDK fetches and caches public signing keys from `/zauth/jwks.json`; verification does not call the validation endpoint on every request. The expected issuer remains `ziqx.cc`, including in local development.

Verified claims include `sub` (Ziqx open ID), `aud`, `azp`, `iss`, `iat`, and `exp`. For Developers apps, `email` is empty without the `email` scope, `username`/`fullname` are empty without `profile`, and `phone` is always empty. Do not treat those fields as guaranteed populated profile data.

`validate(token, { appKey, appSecret })` is deprecated. Without both credentials it resolves to `false`; new integrations should use `verify(token, { appKey })`.

## Local development

Both classes accept `baseUrl: "http://localhost:3000"` for a local core-web. The default is `https://ziqx.cc`, not the Developers API origin. Use the same host configuration for login and token operations. Register the callback URL exactly; plain HTTP redirects are permitted only for `localhost`. Set session-cookie security attributes to suit the app's HTTPS production deployment and local development environment.
