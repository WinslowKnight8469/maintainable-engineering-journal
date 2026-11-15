# Support SMS Event Notification Alerts: Delivery Status Before Another Interruption

Route every contact form to a support queue; reserve SMS event notification alerts for the subset that needs an interruption, and poll delivery status before sending again. An explicit category map and an opt-in escalation flag are the least complex starting point for this B2B SaaS workflow. They also give a small team a useful baseline before adding model-based classification.

Short answer: a Node.js worker should poll the existing delivery status before permitting a resend, cancel future submissions when support acknowledges the ticket, and recheck rate limits and destination permissions for each attempt. Keep queue assignment independent of those decisions. A notification held for review must still leave a usable ticket in the right queue.

The integration decision is where to spend effort first: getting a contact into the correct work queue, or automating another interruption. Start with the queue. The implementation below makes escalation a separate permission, using US and French destinations as illustrative configuration for US and EU coverage. It does not treat those two entries as a regional compliance policy.

## When should SMS event notification alerts wait for delivery status?

The form supplies a category and message; trusted application configuration supplies the queue, recipient, and escalation permission. Persist the ticket first, then create notification work for eligible tickets. A worker submits a short ticket reference, retains the attempt identifier, and polls that attempt later. Acknowledgment disables further submissions even if delivery observation continues.

Keep the ticket.

For this example, a billing contact can request escalation, an access contact can remain in its queue without SMS, and an unrecognized category goes to triage. These are proposed product rules, not classifications that a messaging service makes. The public form never chooses a phone number or country. A free-text request to “send this to another number” has no authority over the configured recipient.

That boundary also limits the initial LLM integration. The tool-use guide listed in References describes defining tools for model interaction; it does not establish permission to contact a recipient. If classification later uses a model, keep its output to a proposed queue label and validate that label against application configuration. Recipient selection and interruption permission remain application decisions.

## Run the interruption policy without a messaging account

This TypeScript example is a synchronous local simulation for a TypeScript-capable Node.js runtime. It needs no external packages or credentials. Its clock values, two-attempt ceiling, and 60-second destination cooldown are illustrative application settings. Call `poll` and `send` from a scheduler in a deployed implementation; these functions do not schedule themselves.

```ts
type Country = "US" | "FR";
type Status = "pending" | "delivered" | "failed" | "unknown";
type Recipient = { key: string; country: Country; enabled: boolean };
type Ticket = {
  id: string; queue: string; recipient: Recipient;
  escalate: boolean; acknowledged: boolean; attempts: number;
  status?: Status; attemptId?: string; review: boolean;
};
type Transport = {
  submit(reference: string, recipient: Recipient): string;
  status(id: string): Status;
};

const recipients = {
  billing: { key: "billing-on-call", country: "US", enabled: true },
  access: { key: "access-on-call", country: "FR", enabled: true },
  triage: { key: "triage-on-call", country: "US", enabled: true }
} satisfies Record<string, Recipient>;
const allowed = new Set<Country>(["US", "FR"]);
const nextSlot = new Map<string, number>();

function accept(id: string, category: string, urgent: boolean): Ticket {
  const queue = category === "billing" || category === "access"
    ? category : "triage";
  return {
    id, queue, recipient: recipients[queue], escalate: urgent,
    acknowledged: false, attempts: 0, review: false
  };
}

function send(t: Ticket, now: number, transport: Transport): void {
  if (!t.escalate || t.acknowledged || t.review) return;
  if (t.status && t.status !== "failed") return;
  if (!t.recipient.enabled || !allowed.has(t.recipient.country) ||
      t.attempts >= 2) {
    t.review = true;
    return;
  }
  if (now < (nextSlot.get(t.recipient.key) ?? 0)) return;
  nextSlot.set(t.recipient.key, now + 60_000);
  t.attempts += 1;
  t.status = "unknown";
  try {
    t.attemptId = transport.submit(t.id, t.recipient);
    t.status = "pending";
  } catch {
    t.review = true; // Reconcile before authorizing another submission.
  }
}

function poll(t: Ticket, transport: Transport): void {
  if (!t.attemptId || t.status === "delivered" || t.status === "failed") return;
  try { t.status = transport.status(t.attemptId); }
  catch { /* Keep the previous observation; do not authorize a resend. */ }
}

const observed = new Map<string, Status>();
let serial = 0;
const transport: Transport = {
  submit() {
    const id = `attempt-${++serial}`;
    observed.set(id, "pending");
    return id;
  },
  status(id) { return observed.get(id) ?? "unknown"; }
};

const ticket = accept("contact-42", "billing", true);
send(ticket, 0, transport);
observed.set(ticket.attemptId!, "failed");
poll(ticket, transport);
send(ticket, 10_000, transport); // Cooldown prevents this resend.
send(ticket, 60_000, transport); // Confirmed failure permits attempt two.
ticket.acknowledged = true; // Cancel future application submissions.
observed.set(ticket.attemptId!, "delivered");
poll(ticket, transport); // Observation can continue after acknowledgment.
send(ticket, 120_000, transport);
console.log(ticket.queue, ticket.attempts, ticket.status);
// billing 2 delivered
```

The example deliberately separates three operations: assigning work, requesting an interruption, and observing an attempt. That separation makes the adapter small. Its implementation must translate documented remote outcomes into the four local states without turning an unfamiliar status into a confirmed failure. The demonstration transport is an in-memory simulator, not a representation of any provider API.

Its main limitation is durability: it doesn't support recovery across restarts or coordination between workers. Objects and maps are sufficient to inspect this policy in one process; a deployed worker needs durable ticket and attempt records, atomic submission claims, and shared limiter reservations. Commit the ticket and notification intent together so a worker can pick up committed work. Keep those storage requirements visible when estimating integration effort: the short transport interface is only part of the job.

## Every additional interruption spends permission again

The retry path needs a fresh decision. In the example, only an observed `failed` state opens that path. Pending and unknown outcomes preserve the existing attempt. A polling exception leaves the last observation intact; an exception during submission sends the ticket to review because the remote acceptance outcome may be uncertain. The trade-off is slower automatic recovery in exchange for fewer speculative duplicate interruptions. This policy isn't suitable for unattended recovery unless the deployment also supplies a verified reconciliation path.

Wait for evidence.

There are two different limits here. Two attempts bound escalation for one ticket, while the shared 60-second slot bounds submissions across tickets targeting the same configured recipient. Ten new tickets do not each receive an independent destination slot. Under contention, this example leaves scheduling order to the caller; a production scheduler must decide how waiting tickets share capacity and how old work becomes visible to an operator. Neither numeric limit claims to match a carrier limit or a statutory requirement.

Recheck the recipient before a retry. Disabling the French configuration entry, or removing `FR` from the application allowlist, prevents another submission even when the first attempt failed. Treat each actual destination and message purpose as a separate policy review. One EU country is not a substitute for the rest, and an application allowlist alone does not establish permission to send in either region.

Cancellation needs a similarly narrow promise. Here it means “stop future application submissions.” It does not recall an accepted message. In a concurrent deployment, define the atomic submission claim as the boundary: acknowledgment blocks unclaimed work, while an already claimed attempt may proceed. Retain any later delivery observation against that attempt so support can distinguish cancellation of future work from the outcome of an earlier send. A remote cancellation operation, if documented by the selected transport, needs its own recorded result.

## Budget observation work separately from message attempts

Polling needs a deadline and a schedule of its own. The 60-second value in the example constrains submissions only. Before deployment, persist a next-poll time, impose a bounded observation window, and move unresolved attempts to review when that window ends. Expiry should not manufacture a failed outcome. This gives the operator a finite amount of scheduled observation work without silently authorizing a duplicate.

Count status requests separately.

For a small team, the useful operating budget includes submission attempts, polling calls, unresolved outcomes, and human review time. Track the oldest unresolved attempt alongside those counts. A quiet worker can still have work stranded in review, so a low submission count is not sufficient evidence that support received the intended interruption. Record ticket ID, queue, attempt number, observed state, and the reason a submission was suppressed. Avoid copying contact-form text into routine worker logs.

Keep the SMS payload to a ticket reference and the minimum context your support process requires. Support can read the full contact in the authenticated queue. This is a proposed data-minimization choice, with a real usability cost: the recipient must open the ticket to understand the request. Test that interaction with the people answering the queue before adding richer messages, more retries, or automated classification.

Model tokens belong in another counter if classification is introduced. An explicit category requires no model request in this implementation. Compare a proposed classifier against that baseline using misrouted contacts, ambiguous categories, and the effort required to inspect a routing decision. Extra intelligence has to earn its place in the contact workflow.

## Release the queue path before widening escalation

Before enabling SMS for a queue, verify that a non-escalated form still creates the correct ticket and that an unknown category remains visible in triage. Replay acknowledgment before a submission claim, a failed attempt during cooldown, and a destination disabled between attempts. Then exercise polling exceptions and uncertain submissions. The expected result is a retained ticket with an explainable notification decision, including cases that need review.

Roll out with durable claims and shared limits in place. Give the review queue an owner, expose its oldest item, and verify restart recovery before widening the eligible ticket set. Persist attempt history rather than replacing it with only the latest identifier, as this compact simulation does. An operator needs both attempts when explaining a resend.

Email fallback would be another integration with its own requirements. The sender guidelines in References describe email authentication and sending practices; they do not establish SMS delivery behavior. Introduce fallback only after defining its permission, suppression, and acknowledgment rules. Otherwise an unresolved SMS attempt can turn into two channels interrupting the same person without a clear decision.

The release criterion is concrete: support receives the contact in the right queue, and every additional interruption has a recorded reason. Expand automation when those records make the next implementation decision easier.

## Further reading

References:

- Tool definitions and model interaction: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
- Email authentication and sender practices, relevant when evaluating a separate email fallback: https://support.google.com/a/answer/81126
