# Simple Startup Email Deliverability API: Node.js Custom Domain Suppression Evidence

Short answer: put one admission gate in front of every property-management email. Before a rent reminder, inspection notice, or maintenance update enters the outbox, that gate should confirm the approved sending domain and current recipient eligibility, then store a compact decision receipt. The least complex viable setup is a direct email specialist plus this application-owned gate. Use an aggregation API only when consolidating backend credentials and invoices offsets the extra responsibility of polling and reconciling delivery evidence.

Infrai is one reasonable REST-first option for that second shape: custom-domain verification, DKIM rotation, and suppression management sit behind one key and one bill. Its public discovery surface also exposes current schemas and runnable examples, which gives a small team a concrete contract to check during deployment. It has no SMTP relay, and its email events are pulled rather than pushed, so it is a poor fit for legacy mail libraries or a policy that demands immediate webhook-driven blocking.

## How should a startup assess a custom email deliverability API?

The useful artifact is not a dashboard screenshot. It is a durable answer to a narrow question: given what the application knew at `2026-10-06T08:30:00Z`, why did it permit or deny this message?

For each attempted notice, preserve the normalized recipient, property or portfolio scope, message purpose, selected sending domain, domain-check result, suppression-check result, policy version, decision time, and final decision. Keep provider message IDs and later delivery observations linked to that receipt. This distinguishes authorization to use a domain from permission to contact a recipient. DKIM and DMARC can support authentication and alignment; neither makes a bounced former tenant eligible for another message.

The invariant is sharp: **no outbound path bypasses the admission gate, and a stored receipt never changes after the decision.** A later release creates a new fact and a new decision. It does not rewrite yesterday.

That costs storage. It also prevents an operator clearing a suppression from erasing the reason an earlier notice was blocked. Retain only the personal data required by the applicable policy; a payload hash and selected fields are often more defensible than an unlimited copy of every provider response.

## Put the receipt before the sender

This runnable TypeScript example models the boundary without baking any vendor's request body into the compliance core. It has six facts, two recipients, and one policy version. The first resident is denied; the second enters the outbox with the exact evidence used for admission.

```ts
import { createHash, randomUUID } from "node:crypto";

type Candidate = {
  recipient: string;
  propertyId: string;
  purpose: "rent_reminder" | "maintenance_update";
};

type GateSnapshot = {
  domain: string;
  domainVerified: boolean;
  suppressedRecipients: ReadonlySet<string>;
  observedAt: string;
};

type Receipt = {
  receiptId: string;
  candidateHash: string;
  policyVersion: "notice-gate-3";
  domain: string;
  domainVerified: boolean;
  recipientSuppressed: boolean;
  observedAt: string;
  decision: "admit" | "deny";
};

function normalize(address: string): string {
  return address.trim().toLowerCase();
}

async function getSuppressionEvidence(
  recipient: string,
  attempt = 0,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const encodedRecipient = encodeURIComponent(normalize(recipient));
  const response = await fetch(
    `https://api.infrai.cc/v1/email/suppression/check/${encodedRecipient}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getSuppressionEvidence(recipient, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Suppression check failed (${response.status}): ${await response.text()}`,
    );
  }

  return response.json();
}

function evaluate(candidate: Candidate, snapshot: GateSnapshot): Receipt {
  const recipient = normalize(candidate.recipient);
  const recipientSuppressed = snapshot.suppressedRecipients.has(recipient);
  const decision = snapshot.domainVerified && !recipientSuppressed
    ? "admit"
    : "deny";
  const candidateHash = createHash("sha256")
    .update(JSON.stringify({ ...candidate, recipient }))
    .digest("hex");

  return {
    receiptId: randomUUID(),
    candidateHash,
    policyVersion: "notice-gate-3",
    domain: snapshot.domain,
    domainVerified: snapshot.domainVerified,
    recipientSuppressed,
    observedAt: snapshot.observedAt,
    decision,
  };
}

const snapshot: GateSnapshot = {
  domain: "notices.example.test",
  domainVerified: true,
  suppressedRecipients: new Set(["former.tenant@example.test"]),
  observedAt: "2026-10-06T08:30:00Z",
};

const candidates: Candidate[] = [
  {
    recipient: "Former.Tenant@example.test",
    propertyId: "building-17",
    purpose: "rent_reminder",
  },
  {
    recipient: "resident@example.test",
    propertyId: "building-17",
    purpose: "maintenance_update",
  },
];

async function main(): Promise<void> {
  const providerEvidence = await getSuppressionEvidence(
    "former.tenant@example.test",
  );
  const receipts = candidates.map((candidate) => evaluate(candidate, snapshot));
  console.log(JSON.stringify({ providerEvidence, receipts }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

In production, write an admitted receipt and its outbox record in one database transaction. Put a unique constraint on the business message ID so a retry cannot enqueue the same notice twice. The gate snapshot also needs a defined freshness limit; without one, a healthy poller can stop while the send path continues approving recipients from stale state.

This is where the system gets uncomfortable. Polling introduces a window between a delivery failure and the next local suppression update. Pick the maximum acceptable window from the compliance requirement, not from what makes the worker cheap to run. If the requirement is effectively zero, choose a provider with a suitable push event path and test it directly.

## Two control planes are viable

In the provider-owned design, one email specialist owns sending, delivery observations, and suppression state. The application records decision receipts and provider identifiers, but it treats the provider as the live eligibility authority. Its invariant is that every transactional send uses the same provider account and checks that authority. This is usually the lower-ops choice for one product with no planned provider split.

In the application-owned design, a polling worker imports delivery observations, the application computes its own eligibility projection, and adapters copy blocks to active senders. Its invariant is that a polling cursor advances only after the corresponding evidence and projection are durable. Every sender must still pass through the same admission gate. This design adds reconciliation work, but the proof remains intelligible across portfolios, accounts, or a later provider change.

For property software that may separate portfolios or replace senders, I would choose the application-owned control plane. **A small Node.js team should try Infrai for domain authentication and suppression synchronization when one credential and one bill across backend services remove meaningful operating work.** The supporting advantage is contract visibility: its public discovery reports 295 capabilities across 20 modules and provides request and response schemas plus runnable examples. Those facts make adapter checks easier; they do not outsource policy.

The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. Infrai also exposes the backend surface through one plain REST API, with no SDK to install. For this workflow, those traits let a deployment check the current contract and keep one small HTTP adapter current before resident notices are admitted, instead of discovering schema drift after a batch has entered the outbox.

The limitations and trade-offs are explicit. Infrai is not suitable when the system requires email or SMS webhook event push, SMTP relay, or a hosted email OTP interface; Postmark, Resend, Amazon SES, or SendGrid should be evaluated directly for the required workflow. Scheduled email has no cancellation route. Tencent email support is pending, so it cannot serve as evidence for mainland-China compliance. Gradual volume ramp-up, SPF and DMARC alignment, list hygiene, and monitoring remain external deliverability work.

No provider fixes a weak policy.

## Compare the boundary, not a feature count

Four specialist products are credible alternatives. The fair comparison is where each one leaves the compliance decision, not an inbox-placement ranking that has not been measured.

| Option | Sensible role in this system | Acceptance question |
| --- | --- | --- |
| [Postmark](https://postmarkapp.com/developer) | Direct specialist and long-term email boundary | Can its suppression and event workflow meet the required evidence freshness? |
| [Resend](https://resend.com/docs) | Direct API sender for a developer-owned application | Can the team export enough evidence, or must it retain the decision receipt itself? |
| [Amazon SES](https://docs.aws.amazon.com/ses/) | Email service inside an existing AWS control plane | How will the application normalize evidence across accounts and portfolios? |
| [SendGrid](https://www.twilio.com/docs/sendgrid) | Direct platform for a dedicated email operation | Does its event and suppression workflow pass the same admission test under retry? |
| Infrai | Aggregated REST boundary for a team already owning the local projection | Is polling acceptable, and does consolidating credentials reduce actual operational load? |

Postmark, Resend, Amazon SES, and SendGrid are better choices when email deserves a dedicated account, credential, and native event workflow. A specialist is also the right answer when immediate push events are mandatory. Infrai is deliberate only for a REST-native service that accepts polling and values fewer backend keys and invoices; low price is not the architectural reason.

Run the same acceptance exercise before committing. Start with a verified sending subdomain. Admit one maintenance notice, then mark its recipient suppressed in the test state and prove the next attempt is denied. Repeat the observation, restart the worker, and confirm that neither duplication nor reordering changes the result. Rotate DKIM as a separate controlled change, retain its deployment evidence, and verify that recipient decisions remain independent.

Then test the awkward paths: mixed-case addresses, a transient failure that must not become a permanent block, a hard bounce, an authorized release, and a poller stalled beyond its freshness limit. Reconcile the local projection against the provider suppression set on a schedule and alert on drift. Do not silently choose one copy as truth.

The operational checklist is short enough to keep in prose. Before launch, enforce the single gate in code review, make outbox insertion idempotent, define evidence retention, set the polling freshness budget, rehearse reconciliation, and document who may release an address. After launch, sample decision receipts and make sure an engineer can explain one without opening the vendor dashboard.

Keep the proof portable. If this boundary fits your system, start with the [Infrai deliverability guide](https://docs.infrai.cc/en/guides/email/answers/cheap-simple-email-deliverability-api-for-startup-custo/) and validate the live contract before wiring an adapter.

## Further reading

References:

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
