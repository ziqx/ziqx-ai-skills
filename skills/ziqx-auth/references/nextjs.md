# Next.js App Router: Sign in with Ziqx

Use these server routes with `@ziqx/auth`. Set `ZAUTH_APP_KEY`, `ZAUTH_SECRET`, and `ZAUTH_REDIRECT_URL` on the server. Register `ZAUTH_REDIRECT_URL` under your app's Auth redirect URLs; it must point to the callback route and match exactly. Set `ZAUTH_BASE_URL` only when targeting a local core-web.

The examples assume HTTPS in production. In local development the cookies omit `Secure`; use an explicitly registered `http://localhost` callback. Adapt session lifetime and destination to the app's existing session design. Temporary cookies support one sign-in attempt at a time per browser; a concurrent-flow implementation should use a server session keyed by state.

## Start sign-in

```ts
// app/auth/login/route.ts
import { NextResponse } from "next/server";
import { ZAuthClient, createPkce } from "@ziqx/auth";

export async function GET() {
  const { verifier, challenge } = await createPkce();
  const state = crypto.randomUUID();
  const auth = new ZAuthClient({
    authKey: process.env.ZAUTH_APP_KEY!,
    baseUrl: process.env.ZAUTH_BASE_URL,
  });

  const response = NextResponse.redirect(auth.getLoginUrl({
    redirectUrl: process.env.ZAUTH_REDIRECT_URL!,
    codeChallenge: challenge,
    codeChallengeMethod: "S256",
    state,
  }));
  const cookie = {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax" as const,
    path: "/",
    maxAge: 600,
  };
  response.cookies.set("zauth_verifier", verifier, cookie);
  response.cookies.set("zauth_state", state, cookie);
  return response;
}
```

Link the login button to `/auth/login`. The verifier is kept in an HttpOnly cookie; only its S256 challenge goes to the authorization URL.

## Handle the callback

```ts
// app/auth/callback/route.ts
import { NextRequest, NextResponse } from "next/server";
import { ZAuthTokenService } from "@ziqx/auth";

export async function GET(request: NextRequest) {
  const code = request.nextUrl.searchParams.get("code");
  const state = request.nextUrl.searchParams.get("state");
  const expectedState = request.cookies.get("zauth_state")?.value;
  const verifier = request.cookies.get("zauth_verifier")?.value;

  function reject(reason: string) {
    const url = new URL("/login", request.url);
    url.searchParams.set("error", reason);
    const response = NextResponse.redirect(url);
    response.cookies.delete("zauth_verifier");
    response.cookies.delete("zauth_state");
    return response;
  }

  if (!code || !state || !expectedState || !verifier || state !== expectedState) {
    return reject("state");
  }

  try {
    const result = await new ZAuthTokenService({
      baseUrl: process.env.ZAUTH_BASE_URL,
    }).getAuthToken({
      authAppKey: process.env.ZAUTH_APP_KEY!,
      authSecret: process.env.ZAUTH_SECRET!,
      code,
      codeVerifier: verifier,
      redirectUri: process.env.ZAUTH_REDIRECT_URL!,
    });
    if (!("access_token" in result)) return reject(result.error);

    const response = NextResponse.redirect(new URL("/", request.url));
    response.cookies.set("session", result.access_token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax",
      path: "/",
    });
    response.cookies.delete("zauth_verifier");
    response.cookies.delete("zauth_state");
    return response;
  } catch {
    return reject("temporarily_unavailable");
  }
}
```

The callback consumes temporary cookies on success and failure. The token stays in an HttpOnly session cookie. The example uses a fixed post-login destination; preserve an existing app's validated return-path handling when adapting it.

## Read the session

```ts
// lib/session.ts
import { cookies } from "next/headers";
import { ZAuthTokenService } from "@ziqx/auth";

const tokenService = new ZAuthTokenService({
  baseUrl: process.env.ZAUTH_BASE_URL,
});

export async function getSession() {
  const token = (await cookies()).get("session")?.value;
  if (!token) return null;
  try {
    return await tokenService.verify(token, {
      appKey: process.env.ZAUTH_APP_KEY!,
    });
  } catch {
    return null;
  }
}
```

Call `getSession()` from the server code that protects the resource. It returns verified claims or `null`; it does not replace your app's own authorization rules. The `await cookies()` form works with the asynchronous Next.js API.
