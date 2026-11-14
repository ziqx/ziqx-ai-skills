# Ziqx AI Skills

Agent skills for integrating the JavaScript and TypeScript SDKs used with [Ziqx Developers](https://developers.ziqx.cc).

## Installation

```bash
npx skills add ziqx/ziqx-ai-skills
```

Select the skills needed for your app, or install one by its skill name:

```bash
npx skills add ziqx/ziqx-ai-skills --skill ziqx-otp-sdk
```

## Available skills

| Skill | npm package | Guidance |
| --- | --- | --- |
| [ziqx-auth-sdk](skills/ziqx-auth/SKILL.md) | `@ziqx/auth` | S256 PKCE sign-in, server-side code exchange, JWT verification, and a Next.js App Router reference. |
| [ziqx-drive-sdk](skills/ziqx-drive/SKILL.md) | `@ziqx/drive` | Server-side upload/delete signing, browser uploads, stored filenames, and local endpoint configuration. |
| [ziqx-otp-sdk](skills/ziqx-otp-sdk/SKILL.md) | `@ziqx/otp` | Server-side India SMS delivery, delivery status, idempotent retries, and the application's code-verification responsibilities. |

## Maintaining the skills

Check the SDK's exported types, implementation, and README when updating an integration. The package-local `agent-skills/` directories in `ziqx-auth-js`, `ziqx-drive-js`, and `ziqx-otp-js` mirror these skill instructions and their references; keep both copies aligned when changing an API or workflow. Runtime code and npm version changes belong in the relevant SDK repository.
