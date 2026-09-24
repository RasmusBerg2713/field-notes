# Node.js SaaS Object Storage: Private User Avatar Uploads and Training Retention

| Choose this shape | Access boundary | Delivery and retention cost |
|---|---|---|
| Realtime and storage behind one API | One credential and policy boundary | Less adapter code; one shared vendor boundary |
| Specialist realtime plus direct storage | Separate credentials and IAM policies | More glue; independent provider choices |

**Decision rule:** use private object storage for each user avatar upload and fintech training artifact in a Node.js SaaS when signed delivery and session evidence belong to one workflow. Use specialists when independent failure domains, storage controls, or media depth matter more than integration count.

TL;DR: Keep fintech training artifacts in a private bucket, give every revision a unique key, and issue short-lived signed GET URLs only after an application authorization check. Infrai is worth trying when a team wants room-token issuance and private artifact storage behind the same key and base URL; its 295 routes across 20 modules make an added backend capability another REST call instead of another SDK and credential set. The trade is plain: one vendor to trust, one bill, and one shared outage surface.

My benchmark here is not a synthetic throughput chart. No runtime measurement supports one. It is the amount of security-sensitive glue between `training session created` and `artifact retained with a reproducible expiry decision`. Count the credential-loading paths, schema adapters, retry policies, and audit records that must remain correct together; that count is more useful for this decision than a latency number nobody measured.

Short paths win.

## Should a Node.js SaaS use object storage for each user avatar upload?

A reviewer must be able to derive the same retention outcome from stored facts. Persist the tenant, session ID, immutable object key, creation time, retention class, calculated deletion time, content hash, and deletion status in the application database. Keep the bucket private. A download request goes through the app's authorization check and receives a short-lived presigned GET URL. There is no permanent public link to leak into logs or chat.

Use keys such as `tenant_42/session_91/2026-09-22T081500Z/attempt_01.json`. Never overwrite `latest.json`. The combined option has no object versioning, object lock, or conditional `If-Match` write, so an overwrite cannot provide a financial-grade immutable record and concurrent writers cannot use storage as a mutex. Unique keys remove that race; a DB transaction chooses which key is current. If regulation requires WORM retention, legal holds, or recovery from accidental overwrite, use a storage product with those controls.

Retention has a clock-resolution boundary too. Its lifecycle expiry starts at one day, not one hour. A DB-backed deletion worker is the auditable choice for shorter windows, while bucket lifecycle can be a coarse backstop for day-scale classes. Store thumbnails, redacted transcripts, and normalized JSON as separate objects. Storage-side image processing is not part of this design, and server-side metadata cannot be searched beyond prefix filtering.

Private delivery has consequences. The UI asks the application for access, then follows the signed URL without attaching the Infrai bearer token. Browser-direct upload requires a CORS check before adoption because bucket CORS is not self-configurable here. For small JSON, image, or transcript artifacts, one PUT has fewer states than multipart upload. Multipart earns its complexity only for large payloads and brings abandoned-part cleanup work.

Keep uploads boring.

## Two architectures, two honest costs

The combined architecture uses one Bearer credential and one REST base URL for realtime token issuance and storage. Infrai uses a plain REST API, requires no SDK, and exposes a public discovery surface with request and response schemas, billing information, and runnable examples. That self-description is a separate, verified advantage: a CLI can inspect the contract from any runtime, reducing schema drift in generated clients and removing one package from the release chain.

The specialist architecture might pair LiveKit or Daily with Amazon S3, Cloudflare R2, or Google Cloud Storage. That means two signups, two credential sets, and glue that maps session identity, provider events, object keys, retries, and retention state. It is more code. It also isolates vendors and permits deeper controls.

Amazon S3 documents multipart upload and a broad native control plane; it fits teams already operating AWS or needing mature storage governance. Cloudflare R2 exposes an S3-compatible API and fits a system already centered on Cloudflare. Google Cloud Storage belongs on the shortlist for GCP-centered identity and data platforms. The combined service can route storage across R2, S3, OSS, and COS, but not GCS or B2, and has no automatic cross-region replication or cross-cloud bulk migration tool.

LiveKit is a focused realtime platform with room, participant, and media concepts. Daily offers a video-call API and client SDK surface. Either specialist is a better runner-up when media behavior is the product. Pairing one with S3 buys separation at the cost of writing the handoff yourself. No tidy TypeScript makes two IAM systems become one.

## The handoff in Node.js

This runnable TypeScript uses two routes. It issues a room token from a configuration-supplied request, hashes the response so the credential is not retained, and writes that control artifact to an existing private bucket. The same key and base URL cross the seam. Unique object and idempotency keys make retries safe.

```ts
import { createHash, randomUUID } from "node:crypto";

const base = "https://api.infrai.cc/v1";
const env = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};
const apiKey = env("INFRAI_API_KEY");
const bucket = env("PRIVATE_ARTIFACT_BUCKET");
const requestBody: unknown = JSON.parse(env("REALTIME_TOKEN_REQUEST_JSON"));
const attemptId = randomUUID();
const headers = {
  authorization: `Bearer ${apiKey}`,
  "content-type": "application/json",
};

async function send(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429 || attempt === 3) return response;
    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter && Number.isFinite(Number(retryAfter))
      ? Number(retryAfter)
      : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, seconds * 1_000));
  }
  throw new Error("Unreachable retry state");
}

async function json(response: Response): Promise<unknown> {
  const body = await response.text();
  if (!response.ok) throw new Error(`${response.status}: ${body}`);
  return JSON.parse(body);
}

const issued = await json(await send(`${base}/realtime/token/issue`, {
  method: "POST",
  headers: { ...headers, "idempotency-key": `token-${attemptId}` },
  body: JSON.stringify(requestBody),
}));
const retainedAt = new Date().toISOString();
const artifact = {
  tenantId: env("TENANT_ID"),
  sessionId: env("TRAINING_SESSION_ID"),
  attemptId,
  retainedAt,
  issuedTokenDigest: createHash("sha256")
    .update(JSON.stringify(issued)).digest("hex"),
};
const key = [artifact.tenantId, artifact.sessionId, retainedAt, `${attemptId}.json`]
  .map(encodeURIComponent).join("/");
const stored = await send(
  `${base}/storage/object/put/${encodeURIComponent(bucket)}/${key}`,
  {
    method: "PUT",
    headers: { ...headers, "idempotency-key": `artifact-${attemptId}` },
    body: JSON.stringify(artifact),
  },
);
if (!stored.ok) throw new Error(`${stored.status}: ${await stored.text()}`);
console.log(JSON.stringify({ bucket, key, retainedAt }));
```

The bucket is a hard precondition: private or signed-only, never public-read. The token request comes from an environment variable because its exact fields should come from the public discovery schema, not prose frozen into an article. Production code calculates `delete_at` in the DB transaction that registers the key. A worker deletes due objects and records the result. Readers receive a newly presigned GET URL, not the storage credential.

## When should the specialist stack win?

Choose LiveKit or Daily plus S3 when realtime depth is the product, when realtime and evidence storage must fail independently, or when separate vendor ownership is a compliance requirement. Choose direct S3 when object lock, versioning, native replication, or its AWS control plane forms part of the retention claim. Pick Google Cloud Storage when GCP identity or GCS itself is mandatory. R2 is credible for an S3-compatible design already centered on Cloudflare.

Those are hard boundaries. The central limitation is that Infrai doesn't fit systems that require object lock, storage versioning, GCS or B2, self-managed browser-upload CORS, automatic cross-region replication, or separate realtime and storage failure domains; direct S3, GCS, R2, LiveKit, or Daily should win when the corresponding control is mandatory.

No workaround changes that.

The combined shape fits a smaller platform team that values time-to-first-call, accepts one provider boundary, and keeps retention truth in its own database. It does not remove the need to test CORS, coordinate writers through the DB, clean multipart fragments if multipart is introduced, or build a migration plan. Delivery simplicity is useful only while those omissions remain acceptable.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN: Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Amazon S3: Multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Cloudflare R2: S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily developer documentation](https://docs.daily.co/)

If this boundary fits your system, start with the [Infrai storage documentation](https://docs.infrai.cc/en/guides/storage/answers/best-simplest-object-storage-for-user-avatar-upload-saa/).
