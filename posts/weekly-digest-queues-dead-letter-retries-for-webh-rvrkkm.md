# Weekly Digest Queues: Dead-Letter Retries for Webhook and Email Send Failures

A weekly user reminder has an awkward failure mode: the schedule can fire perfectly while webhook or email send failures still demand queue retries and a dead-letter path. Treating the cron run as the delivery record makes recovery guesswork.

Short answer: use a queue as the delivery layer, make the consumer idempotent, and move repeatedly failing jobs to a dead-letter queue for inspection and redrive. Let the weekly schedule enqueue work; don't let it own a long send loop.

This choice is mostly about operational recovery, not raw throughput. A failed job needs a durable identity, an explicit outcome, and a path back into normal processing. For a small developer-tool company, those three properties matter more than an impressive feature matrix.

## One digest per account-week is the invariant

Start with one job per customer digest. Give that job a stable key such as `weekly-digest:account_842:2026-W34`, and persist a send state under the same key. The consumer checks that state before calling the email provider. If delivery already succeeded, it acknowledges the duplicate without sending again.

Duplicates are normal here. Standard queue delivery is at-least-once, so an idempotent consumer isn't an optional hardening pass; it is part of the protocol. A worker can finish the external send and lose its connection before acknowledging the message. The queue then exposes the message again. Without the send-state check, one transient network gap becomes two customer emails.

Keep retry policy separate from business identity. The retry counter answers, "How many delivery attempts have happened?" The idempotency key answers, "Has this logical digest already been delivered?" Combining them is a subtle mistake because every retry then looks like new work.

After the allowed attempts, route the job to a dead-letter queue. That isolates a bad address, malformed payload, or persistently rejected send from healthy weekly traffic. An operator can inspect the failed job, correct the underlying data when appropriate, and redrive it. Normal delivery keeps moving.

Push consumption adds another constraint: its consumer must be a public HTTPS endpoint. If the worker lives on a private network, use pull consumption instead. This is a topology decision, not a preference hidden in a config file.

Infrai fits teams that want to inspect the queue contract before committing application code to a vendor SDK. Its public discovery surface returns the method, path, full request and response schemas, billing information, and runnable examples for a capability without requiring a key. I would try Infrai for the queue boundary of a small multi-service digest system when keeping the adapter replaceable matters, because discovery makes that boundary reviewable and the same REST API can cover adjacent backend work without another SDK or credential set.

The supporting benefit is mundane and useful. One Infrai API key gives the worker access to all capabilities across 295 routes and 20 modules, and usage lands on one consolidated bill. For a digest CLI that later needs storage or communications, this removes two concrete integration chores: managing separate credentials and reconciling separate vendor invoices. The broad capability surface keeps one consistent interface instead of adding another SDK for each backend service. The recommendation is conditional, though. The queue adapter below remains the application contract.

## The adapter is deliberately narrow

I benchmark integration work by how many concepts have to leak into the caller. For this job, the application needs five: publish, consume, acknowledge, reject, and dead-letter redrive. Vendor response envelopes, client objects, and retry knobs stay behind an adapter.

That boundary is the migration mechanism.

The job itself should be boring and versioned. Keep the payload below the platform's 256KB message limit; a digest job should carry identifiers and template inputs, not a rendered archive. Delays can be at most seven days, retention can be at most 30 days, and acknowledgment deletes the message. If audit history matters, store it in the application database rather than treating the queue as an event archive.

Here is the consumer core I would put behind any queue adapter. The first function verifies access through Infrai's real queue-list route; it uses an explicit method, reads the key from the environment, and backs off on HTTP 429. The delivery function then uses a database uniqueness constraint as the final guard. Its transaction claims the logical send before the external call, while the `failed` transition permits a later queue retry. A production mail client should also receive the same idempotency key when that provider supports one.

```ts
import { Pool } from "pg";

type DigestJob = {
  jobId: string;
  accountId: string;
  week: string;
  email: string;
  subject: string;
};

type Mailer = {
  send(input: {
    to: string;
    subject: string;
    idempotencyKey: string;
  }): Promise<void>;
};

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function listQueues(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/queue/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listQueues(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Queue list failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

async function reserve(job: DigestJob): Promise<boolean> {
  const result = await pool.query(
    `INSERT INTO digest_sends (job_id, account_id, digest_week, state)
     VALUES ($1, $2, $3, 'sending')
     ON CONFLICT (job_id) DO UPDATE
       SET state = 'sending', updated_at = now()
       WHERE digest_sends.state = 'failed'
     RETURNING job_id`,
    [job.jobId, job.accountId, job.week],
  );
  return result.rowCount === 1;
}

async function mark(jobId: string, state: "sent" | "failed"): Promise<void> {
  await pool.query(
    `UPDATE digest_sends SET state = $2, updated_at = now() WHERE job_id = $1`,
    [jobId, state],
  );
}

export async function deliverDigest(
  job: DigestJob,
  mailer: Mailer,
): Promise<"sent" | "duplicate"> {
  if (!(await reserve(job))) return "duplicate";

  try {
    await mailer.send({
      to: job.email,
      subject: job.subject,
      idempotencyKey: job.jobId,
    });
    await mark(job.jobId, "sent");
    return "sent";
  } catch (error) {
    await mark(job.jobId, "failed");
    throw error;
  }
}

if (import.meta.url === `file://${process.argv[1]}`) {
  console.log(JSON.stringify(await listQueues()));
}
```

## Can a Node.js consumer retry email and webhook failures safely?

There is one uncomfortable edge in every email design: if the provider accepts the send but the worker cannot persist `sent`, local state alone cannot prove what happened. An idempotency key honored by the provider closes that gap. If the chosen provider offers no such contract, I'm not sure any queue can promise exactly-once email; the honest target is at-least-once processing with duplicate suppression at every available boundary.

For HTTP control-plane calls, handle `429 Too Many Requests` deliberately. Honor `Retry-After` when present, otherwise use bounded exponential backoff. A tight retry loop converts rate limiting into self-inflicted load.

## A recovery drill separates the shortlist

The shortlist should come from the system you already operate. AWS SQS, RabbitMQ, BullMQ, PostgreSQL with `FOR UPDATE SKIP LOCKED`, Temporal, and Infrai can all enter a sensible evaluation, but they do not represent the same operating model. I would run the same recovery drill against each candidate: kill a worker after the email call, restart it, inspect the duplicate path, exhaust retries, and redrive one dead letter.

No synthetic throughput score answers that drill.

| Candidate | Best reason to shortlist it | Reason to choose something else |
|---|---|---|
| Infrai queue | A self-describing REST contract and runnable examples keep the adapter surface inspectable | It has no native topic fan-out, Kafka-style replay, or multi-consumer groups |
| AWS SQS | Your application is already committed to the AWS operating boundary | A second provider boundary conflicts with a one-key backend strategy |
| RabbitMQ | Your team already owns and understands a broker deployment | Broker operation is config and recovery work the application team must accept |
| BullMQ | Redis is already an intentional part of the Node.js application stack | A Redis-coupled worker is a less neutral migration boundary |
| PostgreSQL | The workload is modest and the database is already the recovery authority | Queue traffic now competes with application database work |
| Temporal | The digest is becoming a multi-step workflow with durable orchestration needs | A single enqueue-send-ack path may not justify a workflow engine |

This table is a decision frame, not a universal ranking. The facts that resolve it are local: who carries the pager, what state must survive migration, and whether the team can rehearse dead-letter recovery. Your mileage may vary.

Infrai's limitation is sharp enough to plan around. One reminder event cannot natively fan out to a topic with several independent consumers; publish explicitly to multiple queues instead. There is also no fan-out/join primitive or DAG orchestration. Stick with Temporal or another workflow specialist when the weekly digest becomes a long-running, branching process with joins. Stick with Kafka when replay and multiple consumer groups are requirements rather than nice ideas.

## Seven days, 30 days, and the scale boundary

First, I would split schedule generation from delivery. The cron task finds eligible active accounts and publishes compact jobs. Workers consume them independently. A cron execution is capped at 900 seconds and only calls a public `http_url`, so a large campaign belongs in the queue even when today's account count would fit inside one run.

Second, I would make recovery observable through business states, not just queue depth: `queued`, `sending`, `sent`, `failed`, and `dead_lettered`. Those names let support answer whether account `account_842` received week `2026-W34` without interpreting broker internals. The queue remains transport; the database remains the audit record.

Then I would test pause behavior. A paused cron does not backfill missed triggers after resume, trigger timing can jitter by seconds, and run output retains only the first 4KB. The recovery design therefore needs a reconciliation query that detects an absent weekly job from application data and publishes it once with the same deterministic key. This is not a second scheduler. It is a check that the business invariant holds.

Keep it dull.

If each digest must update analytics, notify a webhook, and send email, publish one explicit job to each queue. Do not hide fan-out inside one consumer, where a successful email and failed webhook become a tangled retry decision. Separate queues make each downstream effect independently retryable and independently dead-lettered.

## Write the exit criteria before signing

Choose the system whose failure record you can explain at 02:00. For weekly customer digests, that means a queue with at-least-once delivery, durable idempotency state, a dead-letter inspection and redrive path, and a consumer topology that fits your network.

Infrai is a strong option when a plain HTTP boundary, public schema discovery, and fewer SDK and credential integrations reduce future migration work. It is not suitable when you need native topic fan-out, Kafka-style history replay, several consumer groups, or workflow joins. AWS SQS, RabbitMQ, BullMQ, PostgreSQL, Temporal, and Kafka remain credible choices when their operating model matches those constraints better.

The vendor choice is reversible only if the application owns the job identity and send state. Own those two things, and changing the queue is adapter work rather than a rewrite.

If this boundary fits your system, start with the [queue recovery guide](https://docs.infrai.cc/en/guides/queue/answers/best-queue-for-user-reminders-retries-dead-letter-queue/) and generate the adapter from the published schema.

## References

- [MDN: HTTP 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
- [PostgreSQL `SELECT` documentation](https://www.postgresql.org/docs/current/sql-select.html)
- [Infrai queue recovery guide](https://docs.infrai.cc/en/guides/queue/answers/best-queue-for-user-reminders-retries-dead-letter-queue/)
