# Signature HQ Node.js / TypeScript SDK

Official TypeScript client for the [Signature HQ](https://signaturehq.com.au) e-signature API (v1).

- Full types, ESM and CommonJS builds, zero runtime dependencies
- Global `fetch`: Node.js 18+, and edge runtimes (Cloudflare Workers, Vercel Edge, Deno, Bun) for everything except
  the Node-only file helpers (`uploadFromPath`, `saveSealedDocument`)
- Typed errors, automatic idempotency keys, safe retries with backoff, timeouts and `AbortSignal`
- Webhook signature verification (Web Crypto)

> Not published yet. The package name `@signaturehq/node` and the default base URL
> `https://app.signaturehq.com.au/api/v1` are placeholders until the first release.

## Install

```bash
npm install @signaturehq/node
```

## Quickstart

```ts
import { SignatureHQ } from '@signaturehq/node'; // or: const { SignatureHQ } = require('@signaturehq/node');

const shq = new SignatureHQ(process.env.SHQ_API_KEY!); // "shq_..." live, "shq_test_..." test mode

const envelope = await shq.envelopes.create({
    title: 'Lease agreement',
    signing_order: 'parallel',
});
const document = await shq.documents.uploadFromPath(envelope.id, './lease.pdf');
const signer = await shq.recipients.add(envelope.id, {
    name: 'Jane Citizen',
    email: 'jane@example.com',
    role: 'signer',
});

// Positions are percentages of the page. set() replaces the whole field set.
await shq.fields.set(envelope.id, [
    {
        document: document.id,
        recipient: signer.id,
        type: 'signature',
        page: 1,
        x: 10,
        y: 80,
        width: 30,
        height: 6,
        required: true,
    },
]);

await shq.envelopes.send(envelope.id);

// Later, once envelope.sealed is true:
await shq.envelopes.saveSealedDocument(envelope.id, './signed.pdf');
```

`shq.isTestKey` / `shq.mode` (`'test'` or `'live'`) tell you which kind of key you configured. Test envelopes are not
legally binding, but their invitations are really emailed: use addresses you control. `envelope.livemode` is `false`
for test envelopes.

Objects are returned exactly as on the wire (snake_case), fully typed (`Envelope`, `Recipient`, `Field`, `Document`,
`WebhookEndpoint`, `WebhookDelivery`, `WebhookEvent`...).

### Every endpoint

| Call                                                                                                  | Endpoint                                  |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `envelopes.list({ status, per_page, page })`                                                          | `GET /envelopes`                          |
| `envelopes.all({ ... })`                                                                              | every page, lazily (`for await`)          |
| `envelopes.create(params, { idempotencyKey? })`                                                       | `POST /envelopes`                         |
| `envelopes.retrieve(id)`                                                                              | `GET /envelopes/{id}`                     |
| `envelopes.update(id, params)`                                                                        | `PATCH /envelopes/{id}`                   |
| `envelopes.delete(id)`                                                                                | `DELETE /envelopes/{id}` (drafts)         |
| `envelopes.send(id, { idempotencyKey? })`                                                             | `POST /envelopes/{id}/send`               |
| `envelopes.void(id, reason)`                                                                          | `POST /envelopes/{id}/void`               |
| `envelopes.downloadSealedDocument(id)` / `saveSealedDocument(id, path)`                               | `GET /envelopes/{id}/sealed-document`     |
| `documents.upload(envelopeId, blobOrBytesOrStream, { filename })`, `uploadFromPath(envelopeId, path)` | `POST /envelopes/{id}/documents`          |
| `recipients.add(envelopeId, params)`                                                                  | `POST /envelopes/{id}/recipients`         |
| `recipients.update(envelopeId, recipientId, params)`                                                  | `PATCH /envelopes/{id}/recipients/{rid}`  |
| `recipients.remove(envelopeId, recipientId)`                                                          | `DELETE /envelopes/{id}/recipients/{rid}` |
| `fields.set(envelopeId, fields)`                                                                      | `PUT /envelopes/{id}/fields`              |
| `webhooks.list()`, `create(params)`, `retrieve(id)`, `update(id, params)`, `delete(id)`               | `/webhooks`                               |
| `webhooks.rotateSecret(id)`                                                                           | `POST /webhooks/{id}/rotate-secret`       |
| `webhooks.ping(id)`                                                                                   | `POST /webhooks/{id}/ping`                |
| `webhooks.deliveries(id, { per_page, page })`, `allDeliveries(id)`                                    | `GET /webhooks/{id}/deliveries`           |

Every method takes a last `{ signal?, timeout? }` argument. Envelope create/update accept `title`, `message`,
`signing_order`, `expires_on` (`YYYY-MM-DD`, Australia/Sydney), `reminder_interval_days` (1-30 or null) and
`append_certificate`. Recipients accept `verification_method` (`email_link` by default, `email_otp`, or `sms_otp` with
`phone`). Radio and dropdown fields take `options` (2 to 20 labels).

Uploads accept a `Blob`/`File`, `Uint8Array`/`Buffer`/`ArrayBuffer`, a web `ReadableStream` or a Node.js readable
stream (buffered in memory, refused above the 25 MB server limit). `downloadSealedDocument()` returns
`{ stream, contentType, contentLength, filename }` with a web `ReadableStream`.

## Configuration

```ts
const shq = new SignatureHQ(process.env.SHQ_API_KEY!, {
    baseUrl: 'https://app.signaturehq.com.au/api/v1', // https only (http for localhost)
    timeout: 60_000, // ms per attempt (downloads: until the response headers)
    maxRetries: 2, // 0 disables retries
    maxRetryAfter: 60, // longest Retry-After (s) honoured automatically
    fetch: customFetch, // optional
    appInfo: 'acme-crm/2.1',
    debug: (event) => console.log(event), // Authorization redacted, bodies never included
});
```

The API key is never put in a URL, never logged and hidden from `console.log`/`JSON.stringify`. Redirects are never
followed. The client refuses to start in a browser (the key would be exposed to visitors) unless you pass
`dangerouslyAllowBrowser: true`.

## Errors

Every error extends `SignatureHQError`. API errors (`APIError`) carry `status`, `type`, `message`, `errors` and
`headers`:

| Error                              | When                                                  |
| ---------------------------------- | ----------------------------------------------------- |
| `AuthenticationError`              | 401: invalid, expired or revoked key                  |
| `PermissionError`                  | 403: missing ability or permission                    |
| `NotFoundError`                    | 404 (envelopes of other organisations are always 404) |
| `ConflictError`                    | 409: e.g. editing an envelope that was already sent   |
| `IdempotencyConflictError`         | 409: the same key is still being processed            |
| `ValidationError`                  | 422: `fieldErrors('email')`                           |
| `IdempotencyKeyReusedError`        | 422: the key was used for a different request         |
| `RateLimitError`                   | 429: `retryAfter` seconds                             |
| `ServerError`                      | 5xx                                                   |
| `InvalidRequestError`              | 400 and other 4xx                                     |
| `ConnectionError` / `TimeoutError` | network failure / no response in time                 |
| `RequestAbortedError`              | your `AbortSignal` fired                              |
| `InvalidArgumentError`             | rejected locally before sending                       |

```ts
import { ValidationError } from '@signaturehq/node';

try {
    await shq.recipients.add(envelopeId, {
        name: 'Jane',
        email: 'nope',
        role: 'signer',
    });
} catch (error) {
    if (error instanceof ValidationError)
        console.log(error.fieldErrors('email'));
    else throw error;
}
```

## Idempotency and retries

`envelopes.create()` and `envelopes.send()` always send an `Idempotency-Key` (a random UUID unless you pass one) and
reuse it for their retries, so a retry does not create a second envelope or send twice once the server has answered. Pass your own key to stay safe across
process restarts:

```ts
await shq.envelopes.create(
    { title: 'Order 1234' },
    { idempotencyKey: 'order-1234-envelope' },
);
```

Retries (default 2, exponential backoff with jitter, at most 8 s between attempts):

- network errors, timeouts and 5xx: only for `GET`, `PUT`, `DELETE` and the keyed create/send;
- 429: always (the rate limiter refuses before any work is done), honouring `Retry-After` up to `maxRetryAfter`;
- 409 `idempotency_conflict`: for keyed requests.

Other `POST`/`PATCH` calls (void, add recipient, upload, webhook create, rotate secret, updates) are never retried after
they may have reached the server. A `DELETE` retried after a network error, timeout or 5xx that then gets 404 (the first attempt went through) counts
as success; after a 429 a 404 is reported as usual.

When retries run out, the error carries the key that was sent (`idempotencyKey`): retry later with that same key, never a
new one, or you may create a second envelope. Keys protect against duplicates only for 24 hours. An aborted request is never retried.

The server stores 4xx responses for a key too. If `send` fails with a 422 because the draft is incomplete, fix the
draft and send again with a new key: reusing the old key replays the same 422 for 24 hours.

## Pagination

```ts
const page = await shq.envelopes.list({ status: 'completed', per_page: 100 });
page.data;
page.meta.total;
page.hasMore();
const next = await page.nextPage();

for await (const envelope of shq.envelopes.all({ status: 'completed' })) {
    // every page, fetched lazily; ids already seen are skipped if new envelopes shift the pages
}
```

## Webhooks

Verify every delivery against the **raw** body (not parsed JSON):

```ts
import express from 'express';
import { Webhook, WebhookSignatureError } from '@signaturehq/node';

app.post(
    '/shq-webhook',
    express.raw({ type: 'application/json' }),
    async (req, res) => {
        let event;
        try {
            event = await Webhook.constructEvent(
                req.body,
                req.get('SHQ-Signature'),
                process.env.SHQ_WEBHOOK_SECRET!,
            );
        } catch (error) {
            if (error instanceof WebhookSignatureError)
                return res.sendStatus(400);
            throw error;
        }
        res.sendStatus(204); // answer fast, then process asynchronously, de-duplicating on event.id (SHQ-Event-Id)
    },
);
```

Edge runtimes: `await Webhook.constructEvent(await request.text(), request.headers.get('SHQ-Signature'), secret)`.

- `Webhook.verifySignature(payload, header, secret)` resolves to a boolean instead.
- The timestamp must be within 5 minutes (`{ tolerance: seconds }` to change it).
- Rotating a secret switches deliveries to the new one immediately: pass both while you deploy,
  `[newSecret, oldSecret]`.
- `Webhook.signatureHeader(payload, secret)` builds a header for your own tests.

## Development

```bash
npm ci
npm run typecheck
npm test          # builds, then runs the node:test unit tests
```

The integration smoke test runs against a local SHQ e2e server: from the repository root inside DDEV,
`E2E_PORT=8024 E2E_DB=e2e_k node sdks/integration/run.mjs ts`.
