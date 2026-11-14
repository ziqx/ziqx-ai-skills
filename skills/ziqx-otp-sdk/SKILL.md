---
name: ziqx-otp-sdk
description: Integrate @ziqx/otp in a server-side JavaScript or TypeScript app to deliver SMS codes and read delivery status. Use for Ziqx OTP keys, India-only recipient validation, idempotent retries, and delivery errors; code verification remains the application's responsibility.
---

# Ziqx OTP SDK

Install `@ziqx/otp` for Node.js 18+. It supports ESM and CommonJS with no runtime dependencies. In **Ziqx Developers → app → OTP**, save the app name, enable SMS, and create an API key. The `zotp_…` secret is shown once; keep it in a server environment variable such as `ZIQX_OTP_API_KEY`. It is separate from Auth credentials and the dashboard token.

All SDK calls belong on the server. Do not put the key or an SDK instance in browser/mobile code or `NEXT_PUBLIC_` variables. Connect the SDK to the app's existing verification workflow and authorization/abuse controls.

## API

| API | Behavior |
| --- | --- |
| `new ZOTP(apiKey, { apiUrl? }?)` | Creates the server client. Defaults to `https://api.ziqx.net`. |
| `otp.send({ to, code, idempotencyKey, channel? }, { signal? }?)` | Returns `Promise<OTPSend>` from `POST /v1/otp/send`. |
| `otp.getStatus(sendId, { signal? }?)` | Returns `Promise<OTPSend>` from `GET /v1/otp/sends/{sendId}`. |
| `ZOTPError` | HTTP/response error with `name`, `message`, and numeric `status`. |

Both routes authenticate with `X-OTP-Key`; sends also include `Idempotency-Key`. Do not use a Bearer dashboard token or invent SDK methods for key management, code generation, expiry, or code verification.

## Send and track a verification attempt

```ts
import { ZOTP } from "@ziqx/otp";

const otp = new ZOTP(process.env.ZIQX_OTP_API_KEY!);

export async function deliverCode(
  to: string,
  code: string,
  idempotencyKey: string,
) {
  // These values belong to an already-persisted verification attempt.
  // A retry receives exactly the same recipient, code, and idempotency key.
  const send = await otp.send(
    { to, code, idempotencyKey },
    { signal: AbortSignal.timeout(20_000) },
  );
  return send; // Persist send.id with the application's verification attempt.
}

export async function readDelivery(sendId: string) {
  return otp.getStatus(sendId, { signal: AbortSignal.timeout(20_000) });
}
```

Generate a cryptographically random six-digit code in the application (for example using Node's `randomInt`). Before sending, persist the recipient, code hash, expiry, attempt limit, and idempotency key. Keep any code needed for delivery retries available securely for the attempt's short lifetime. Verification should compare the submitted code against that attempt, enforce expiry/attempt limits, and consume it after success. The SDK only delivers codes; `getStatus()` reports SMS delivery and does not authenticate the user.

## Input rules and retry behavior

- `to`: an Indian mobile number matching `+91` followed by ten digits starting with `6`–`9`. Pass E.164 format, for example `+919876543210`; the SDK does not normalize numbers.
- `code`: a **string** of exactly six ASCII digits, for example `"001234"`. Numeric input loses leading zeroes and is rejected.
- `channel`: optional; only `"sms"` is supported, and it is the default. The message uses the app name saved in Developers: `Your OTP for {app name} is {OTP}.`
- `idempotencyKey`: required, 8–128 printable ASCII characters with no outer whitespace. Generate it once per logical send, persist it, and reuse it with the same recipient/code on retries. Reusing it for a different payload returns HTTP 409.
- The SDK makes **no automatic retries**. A network failure or abort does not establish that an SMS was never submitted; retry with the existing key rather than creating another send.
- Current backend limits are one new send per phone number per minute and five per hour. Rate limits return HTTP 429; do not bypass them with new keys.

## Delivery records and errors

`OTPSend` contains `id`, `appId`, `status`, `phoneLast4`, optional `provider`, `createdAt`, and `deliveredAt`. Timestamps are ISO 8601 strings; `deliveredAt` is `null` until delivery is confirmed. Status lookups accept a send UUID and are scoped to the API key's app.

| Status | Interpretation |
| --- | --- |
| `created`, `reserved`, `sending` | Preparing or submitting the delivery. |
| `accepted` | Provider accepted the message; delivery is unconfirmed. |
| `unknown` | Outcome is uncertain and awaiting reconciliation. |
| `delivered_pending_charge` | Delivery confirmed; billing settlement is pending. |
| `delivered` | Delivery and billing settlement completed. |
| `failed_pending_release` | Delivery failed; credit release is pending. |
| `failed` | Delivery failed and any credit hold has been released. |

Successful HTTP responses return a record even for `unknown` or failure states. Inspect `status` rather than treating a resolved promise or a returned ID as delivery confirmation.

`ZOTPError.message` preserves an API rejection reason. Common HTTP statuses are 400 (invalid request), 401 (invalid key), 403 (SMS disabled), 404 (send missing), 409 (conflict), 429 (rate limit), and 503 (unavailable). Invalid successful responses also throw `ZOTPError` with that response's HTTP status. Input validation throws `TypeError`; native network and cancellation errors propagate unchanged. Preserve the original error when the application's retry or response handling needs its type/status, and avoid logging codes or API keys.

## Configuration

Use `new ZOTP(key, { apiUrl: "http://127.0.0.1:8080" })` for a local Developers backend. `apiUrl` must be an HTTP(S) origin without credentials, a path, query, or fragment. Both methods accept a request `signal`; there is no SDK-imposed timeout. Requests disable caching and reject redirects. The package also exports `DEFAULT_API_URL` and the types `ZOTPOptions`, `SendOTPParams`, `OTPRequestOptions`, `OTPSend`, and `OTPStatus`.
