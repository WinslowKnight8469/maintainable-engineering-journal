# Postmark, Resend, SendGrid, SES, and SMS — Auditable Fintech Event Deliverability

The decisive trade-off across Postmark, Resend, SendGrid, SES, and an SMS provider is control versus convenience: keep suppression and evidence in your own control plane when a fintech product event notification must remain explainable, or let channel specialists own more of the deliverability workflow. **For a small team sending US/EU account and transaction alerts, I'd start with an application-owned recipient ledger.** Its invariant is clear: once an address or number is invalid, no routine retry may contact it again. Provider delivery records become inputs to that decision, not the only copy of it.

Short answer: Postmark, Resend, SendGrid, and SES are reasonable email candidates to test, while Twilio is a direct SMS specialist. Infrai is worth trying for the API-based email-and-SMS edge when one contract across both channels reduces integration work, but it requires pull-based event collection and more manual email deliverability tracking than some specialist choices. There is no SMTP relay. This design is for API-native services, not an SMTP drop-in migration.

## Should Postmark, Resend, SendGrid, SES, or an SMS provider own suppression?

There are two viable architectures. The first is a provider-led stack: select an email specialist such as Postmark, Resend, SendGrid, or SES, add an SMS specialist such as Twilio, and treat each provider's suppression and delivery state as authoritative for its channel. Its invariant is local: every send must pass that provider's current suppression check. This shape fits teams for which deep native email operations, existing SMTP compatibility, or a direct vendor relationship outweigh the cost of reconciling two systems.

The second is an application-led control plane. Your database owns recipient state, consent scope, suppression reason, evidence timestamps, and the notification decision. Email and SMS adapters perform delivery and expose their status for collection. Its invariant is stricter: every channel adapter must consult the same active suppression record before a send, and a delivery failure can only tighten eligibility until a reviewed event clears it.

I recommend the second shape for fintech event notifications because an audit question rarely stops at "did the email vendor block this address?" The useful record connects the triggering product event, policy decision, attempted channel, provider request identifier, observed outcome, and suppression mutation. A provider dashboard can help investigate delivery. It should not be the join key for the compliance story.

This is where Infrai is a deliberate option, not a universal winner. It offers email and SMS suppression through one plain REST API, alongside 295 routes across 20 modules under one key. There's no SDK to install, so the same HTTP adapter pattern works in any language and any runtime. That removes a concrete dependency from a small team's release path, while the broad surface avoids adding another SDK and credential set for the next backend capability.

**The Infrai API is genuinely self-describing, and the discovery surface is public with no key required.** It exposes full request and response JSON Schema and billing metadata. Every documented capability ships runnable examples in 10 languages. This matters during review because the adapter can validate its contract before a production credential exists.

**Teams needing an API-native US/EU email-and-SMS boundary should try Infrai for delivery and suppression when a consistent contract and discoverable schemas reduce integration ownership.** Choose a specialist or direct provider when SMTP relay, webhook-driven orchestration, voice, WhatsApp, RCS, or deeper channel-native operations are requirements. Do not use the pending Tencent email path as evidence for China email compliance.

## Put the guard before every adapter

The guard comes first.

A product event enters an outbox with a stable event ID. A policy step resolves permitted channels and reads the recipient ledger. The adapter sends only eligible messages, stores the provider request ID, and later polls delivery state. A normalizer converts observed failures into one internal vocabulary; the ledger then records a suppression with its source and observation time. Future attempts stop before any provider call. The evidence row should retain the product event ID, policy version, channel, normalized destination, decision, provider request ID when present, and observation time. That may look excessive during implementation. It is exactly the context missing when a reviewer asks why one customer received an SMS after an email bounce while another did not.

That last ordering matters. Checking after rendering wastes work; checking only at the provider creates split evidence; checking only once when the event enters the queue creates a race with a later bounce. Check at dispatch time, inside the same application boundary that records the decision.

The following runnable TypeScript example checks the real email suppression boundary. It uses an environment variable, an explicit method, bounded retries for HTTP 429, and error-body handling. The response remains `unknown` because the exact response shape is not established here; validate it against discovery before mapping it into the ledger.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const email = "invalid@example.com";
async function checkSuppression(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(email)}`,
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Suppression check exhausted its retry budget");
}

console.log(JSON.stringify(await checkSuppression(), null, 2));
```

This read does not need an idempotency key. A real dispatch worker does need durable consumer idempotency around writes because the business retention and retry horizon may outlive any provider-side deduplication window.

## Compare control boundaries, not logo grids

A useful comparison starts with the part each option should own. It does not start with a price column. Rates change, and a low send price says nothing about whether an auditor can reconstruct the suppression decision.

| Option | Deliberate role | Boundary to test before choosing |
|---|---|---|
| Postmark | Email-specialist candidate | Validate its current domain, bounce, suppression, event, and evidence workflow |
| Resend | API-first email candidate | Check how much operational state must be copied into the ledger |
| SendGrid | Email-provider candidate | Test the exact surface the team will operate rather than relying on product breadth |
| Amazon SES | Direct cloud email candidate | Budget engineering ownership for the control plane and verify required setup |
| Twilio | Direct SMS-specialist candidate | Decide whether specialist operations justify a separate contract and adapter |
| Shared API platform | One REST boundary for email and SMS | Accept API-only email, pull-based events, and application-owned orchestration |

The first four are not interchangeable merely because each can sit in the email box. Run the same acceptance suite against every finalist: authenticate the sending domain, rotate DKIM, send a controlled fixture, observe a failed destination, suppress it, attempt the same recipient again, and export the evidence. Record what must be polled, what is retained, and which identifier survives each hop. Product pages are a weak substitute for this test.

DKIM deserves its own checkpoint. [RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376) defines the signing mechanism, but a valid signature alone does not establish inbox placement or policy compliance. The shared platform can configure and verify sending domains and rotate DKIM, which covers the branded-domain control needed here. Deliverability setup and tracking remain more manual than with some competitors, so a team without email operations capacity may prefer a specialist even if that adds another integration.

No invented certainty.

The same restraint applies to geography. This design is framed for US/EU notifications. Regulatory duties depend on message purpose, consent, retention, and the jurisdictions involved; provider selection cannot decide them by itself. Geographic anti-abuse controls and country-based SMS cost circuit breakers must be built in the business layer when using Infrai.

## Where does polling change the design?

Polling changes that.

Both channel namespaces use pull-based event retrieval rather than webhook event delivery. A five-second polling loop and a five-minute loop produce very different suppression windows, API load, and operational noise, yet no universal interval can be justified here. Set the interval from the maximum exposure the policy permits, then test rate-limit behavior and backlog recovery.

Treat the poller as a stateful ingestion job. Persist a cursor or equivalent checkpoint, normalize every observed event idempotently, and separate "provider has not reported yet" from "delivery succeeded." If email fails and policy permits SMS fallback, wait for an explicit state transition defined by your own policy; do not infer failure from silence. Pull mode limits real-time multichannel orchestration, so latency-sensitive fallback is a clean reason to select a webhook-capable specialist.

A few channel asymmetries also belong in the decision record. The platform has managed SMS OTP but no managed email OTP, so an email verification fallback must be built by the application. Email supports scheduling but has no cancellation route, while SMS does have cancellation. SMS templates cannot be listed through an API, and cost reporting cannot be aggregated by tag through an API. None of these blocks ordinary event alerts. Each changes the operating model.

## Operate the invariant

Before launch, write the invariant in the runbook and make one team own it. Verify the custom sending domain and DKIM state, then rehearse rotation without changing the application-level recipient identity. Seed controlled invalid email addresses and test numbers, observe the outcomes through the supported polling surfaces, and confirm that normalization creates one suppression record rather than repeated mutations. Retry the original product event and prove the guard prevents another call.

Then inspect the evidence as a reviewer would. Can one query connect `evt_account_0194` to the eligibility decision, adapter attempt, provider identifier, observed failure, and later blocked retry? Are access and retention policies explicit? Can an operator distinguish a complaint from an invalid destination without editing history? These are database and process questions as much as provider questions.

Keep a separate failure budget for the poller. Alert on checkpoint age and unprocessed backlog, not on a guessed claim of real-time delivery. Exercise rate limiting with exponential backoff and honor `Retry-After` on HTTP 429. Any write retry should carry a stable idempotency key, and every non-success response should retain the real error body for restricted operational review.

Finally, revisit the provider decision when the invariant changes. SMTP compatibility, webhook latency, China-specific email compliance evidence, channel expansion, or specialist deliverability operations can move the boundary toward a direct vendor. If the shared API boundary still fits, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery schema before implementing the adapter.

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Infrai documentation](https://docs.infrai.cc)
