---
name: ziqx-drive-sdk
description: Integrate @ziqx/drive in JavaScript or TypeScript apps for server-side upload and delete signing, browser uploads, and optional image resizing. Use for Ziqx Drive credentials and stored-file handling, not direct S3/R2 access.
---

# Ziqx Drive SDK

Install `@ziqx/drive`. Obtain the drive key and secret from **Ziqx Developers → app → Drive**. Keep both on the server; the browser receives only a signed upload URL. Use the app's existing authorization checks when exposing signing or deletion operations.

## Choose the API

| API | Environment | Result |
| --- | --- | --- |
| `new ZDrive(driveKey, driveSecret, options?)` | Server | Configures Drive credentials and API/core hosts. |
| `drive.generatePutUrl(fileName, folder?)` | Server | `Promise<SignUrlResponse>` with a signed upload URL. |
| `drive.generateDeleteUrl(fileName, folder?)` | Server | `Promise<SignUrlResponse>` with a signed deletion URL. |
| `drive.deleteFile(fileName, folder?)` | Server | Performs the deletion and returns `Promise<UploadResponse>`; can throw. |
| `new ZDriveClient()` | Browser | Configures credential-free upload helpers. |
| `client.uploadFile(uploadUrl, file)` | Browser | Uploads with multipart/form-data; returns `Promise<UploadResponse>`. |
| `client.resizeImage(file, maxWidth?, quality?)` | Browser | Returns `Promise<File>` using Canvas; can throw. |

`SignUrlResponse` is `{ success: boolean, message: string, url?: string }`. `UploadResponse` is `{ success: boolean, message: string, filename?: string, driveid?: string }`. Check `success` and the required optional field before proceeding. Signing and uploading return normalized failure objects, including for network failures; `deleteFile()` and `resizeImage()` can throw and need the caller's error handling.

## Sign on the server

```ts
import { ZDrive } from "@ziqx/drive";

const drive = new ZDrive(
  process.env.ZDRIVE_KEY!,
  process.env.ZDRIVE_SECRET!,
);

export async function signUpload(fileName: string, folder?: string) {
  // Call after the app has authorized this user's upload and target folder.
  const signed = await drive.generatePutUrl(fileName, folder);
  if (!signed.success || !signed.url) throw new Error(signed.message);
  return signed.url;
}

export async function deleteUpload(storedFilename: string) {
  // Look up and authorize the stored file before allowing its deletion.
  const result = await drive.deleteFile(storedFilename);
  if (!result.success) throw new Error(result.message);
  return result;
}
```

Expose these operations through the app's existing server routes or actions. Never instantiate `ZDrive` or send Drive credentials in client code. In Next.js, a Server Action belongs in a file with `"use server"`; the browser helper belongs in client code.

## Upload in the browser

```ts
import { ZDriveClient } from "@ziqx/drive";

export async function uploadFile(file: File, signedUrl: string) {
  const client = new ZDriveClient();
  const uploaded = await client.uploadFile(signedUrl, file);
  if (!uploaded.success || !uploaded.filename) {
    throw new Error(uploaded.message);
  }
  return uploaded.filename; // Save this exact stored name for later deletion.
}
```

Get `signedUrl` from your server first, then pass the actual `File` to `uploadFile()`. The helper constructs `FormData` and posts directly to the signed URL; do not build a JSON upload body or manually set the multipart boundary.

When resizing is appropriate, `await client.resizeImage(file, 1024, 0.8)` returns a new file. The current helper outputs JPEG, so consider transparency and format requirements before using it. Sign using the file you actually intend to upload.

## Stored names and deletion

Upload signing cleans filenames with the exported `cleanUploadName()` helper. Storage also adds a unique suffix. The original name used for signing is therefore not the stored filename.

- Persist the successful upload response's `filename` and use it for deletion.
- Pass the full relative stored name, such as `users/123/photo-a1b2c3d4.jpg`, or its stored basename plus `folder` as a separate argument.
- Do not use the original name, a CDN URL, or clean/sanitize an existing stored name before deletion. Delete signing sends the stored name verbatim.
- Do not supply the folder twice: `deleteFile("users/123/photo-a1b2c3d4.jpg")` and `deleteFile("photo-a1b2c3d4.jpg", "users/123")` target the same file.

## Endpoints and local development

Uploads are signed with `GET https://api.ziqx.net/v1/drive/sign-url`. On HTTP 401 or 404, the SDK tries core's original signing route once for older drives. Other failures, including rate limits, do not trigger that fallback.

Deletes are signed with `DELETE https://ziqx.cc/api/drive/sign-url`, then `deleteFile()` sends `DELETE` to the returned signed URL. Signing uses `x-drive-key` and `x-drive-secret` headers, with `filename` and optional `folder` query parameters.

```ts
import { ZDrive } from "@ziqx/drive";

const drive = new ZDrive(process.env.ZDRIVE_KEY!, process.env.ZDRIVE_SECRET!, {
  apiUrl: "http://127.0.0.1:8080", // Developers API: upload signing
  coreUrl: "http://localhost:3000", // core-web: delete signing and legacy fallback
});
```

`apiUrl` and `coreUrl` are host overrides, not full signing URLs. For a local setup that supports deletion, configure both hosts rather than pointing all operations at Developers.
