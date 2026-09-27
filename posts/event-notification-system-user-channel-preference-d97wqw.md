# Event Notification System: User Channel Preferences for Support Report Delivery

TL;DR: Keep report templates and channel-policy decisions in your application, then treat email and SMS transports as replaceable delivery adapters. For a customer-support report, generate the attachment once, evaluate the recipient's current channel preferences and suppression status immediately before each send, and record the decision separately from the provider response. This prevents a queued job from turning an old preference snapshot into an unwanted message.

The simplest approach looks attractive: render provider-hosted templates, enqueue one email job and one SMS job, and let each transport decide what to do. It fails the evaluation constraint that matters here: one opt-out must take effect consistently even when jobs wait in different queues. The chosen design owns the template, policy, and audit vocabulary in one place. Providers receive rendered content only after the local policy check passes.

## How should a Node.js event notification system enforce user channel preferences?

A support-report notification has 3 distinct artifacts: the generated report, the email that carries it as an attachment, and the SMS that says the report is ready. A user may allow one channel and reject the other. A destination may also appear on a suppression list after a delivery failure or a direct opt-out. Those are policy inputs, not template variables.

If channel logic lives inside two remote templates, the application cannot explain one decision with one vocabulary. Copy and policy can also drift: an email template may imply that SMS will follow, while the SMS path has already been suppressed. Owning the templates beside the event schema keeps the promise, attachment metadata, and routing rule reviewable in the same change.

The boundary should stay narrow. The application decides `send` or `suppress`; an adapter translates a permitted message into a transport request. This is a deliberate trade-off. Local rendering adds code and deployment responsibility, but it avoids binding business rules to a transport's template model. For a small team, that is usually the cheaper kind of complexity because it can be tested without network calls.

Policy stays local.

## One policy check at the last responsible moment

Store preferences by user and channel, but store suppression by normalized destination and channel. They answer different questions. A preference says what an authenticated user selected. A suppression entry says that a particular address or phone destination must not be used, regardless of which user record currently points to it.

Do not copy `emailEnabled: true` into the queued payload and trust it later. Queue identifiers and immutable report metadata; read mutable consent state when the worker is ready to send. Consider a report event that creates 2 jobs at 09:00: email is picked up immediately, the user opts out of SMS at 09:01, and the SMS worker starts at 09:03. A preference snapshot stored in the job sends the text. A fresh lookup suppresses it. This timeline is illustrative, not a benchmark, but it exposes the exact race the boundary must resolve. There is still a smaller race between the final read and the transport call, so make the policy decision an explicit, timestamped record and ensure opt-out processing is idempotent. I'd pay for that extra policy read; fast cancellation matters more than shaving one database operation from a background job.

Here is the focused shape. The repositories and transports are interfaces on purpose; database and delivery choices do not leak into the rule.

```ts
type Channel = "email" | "sms";
type Decision =
  | { kind: "send"; destination: string }
  | { kind: "suppress"; reason: "preference" | "suppression-list" | "missing-destination" };

type ReportReady = {
  eventId: string;
  userId: string;
  reportId: string;
  reportName: string;
};

interface PolicyStore {
  destination(userId: string, channel: Channel): Promise<string | null>;
  channelEnabled(userId: string, channel: Channel): Promise<boolean>;
  isSuppressed(channel: Channel, destination: string): Promise<boolean>;
  recordDecision(eventId: string, channel: Channel, decision: Decision): Promise<void>;
}

interface Renderer {
  email(event: ReportReady): Promise<{ subject: string; html: string; attachment: Uint8Array }>;
  sms(event: ReportReady): Promise<{ text: string }>;
}

interface Delivery {
  email(to: string, message: Awaited<ReturnType<Renderer["email"]>>): Promise<void>;
  sms(to: string, message: Awaited<ReturnType<Renderer["sms"]>>): Promise<void>;
}

async function decide(
  store: PolicyStore,
  event: ReportReady,
  channel: Channel,
): Promise<Decision> {
  if (!(await store.channelEnabled(event.userId, channel))) {
    return { kind: "suppress", reason: "preference" };
  }

  const destination = await store.destination(event.userId, channel);
  if (!destination) return { kind: "suppress", reason: "missing-destination" };
  if (await store.isSuppressed(channel, destination)) {
    return { kind: "suppress", reason: "suppression-list" };
  }
  return { kind: "send", destination };
}

async function deliverReport(
  event: ReportReady,
  store: PolicyStore,
  renderer: Renderer,
  delivery: Delivery,
): Promise<void> {
  for (const channel of ["email", "sms"] as const) {
    const decision = await decide(store, event, channel);
    await store.recordDecision(event.eventId, channel, decision);
    if (decision.kind === "suppress") continue;

    if (channel === "email") {
      await delivery.email(decision.destination, await renderer.email(event));
    } else {
      await delivery.sms(decision.destination, await renderer.sms(event));
    }
  }
}
```

The example intentionally renders after authorization. That ordering avoids generating an attachment for an email that policy will discard. In a real worker, `eventId` plus channel should also identify one logical delivery attempt so a retry cannot create an unbounded series of sends. The exact persistence mechanism depends on the queue and database; the invariant does not.

Check again at dispatch.

## Make opt-out a write path, not copy

An unsubscribe link or SMS stop request must update durable policy state before any acknowledgement is treated as complete. Normalize the destination using the same rules used during lookup, insert the suppression record idempotently, and preserve the source and effective time for audit. Do not make removal from a marketing list the only action if transactional report notices share the destination: channel and message-purpose rules need an explicit model.

Keep acknowledgement language owned by the application too. For SMS, carrier and industry requirements affect opt-out handling; CTIA publishes messaging interoperability and compliance best practices. The implementation should be reviewed against the rules applicable to the program and jurisdiction rather than assuming one generic consent flag covers every message.

A preference update and a suppression event can arrive in either order. Suppression wins during dispatch. Re-enabling a profile checkbox must not silently erase a destination-level suppression entry; require a separate, auditable consent action appropriate to the channel.

The precedence is short enough to state plainly:

| Current state | Dispatch decision |
| --- | --- |
| Channel disabled | Suppress for preference |
| Destination suppressed | Suppress for destination |
| Destination missing | Suppress for missing destination |
| Enabled, present, and not suppressed | Send through the channel adapter |

## Test decisions before testing transports

Most useful tests never contact a provider. Use a table of policy inputs and assert the decision and reason: email enabled with no suppression sends; SMS disabled suppresses; an enabled channel with a suppressed destination still suppresses; a missing destination does not throw or fall through to another channel. Then add a concurrency test in which a job is queued, the destination is suppressed, and the worker runs. The expected result is zero transport calls.

Test template ownership as a contract. Given one report event, assert the email subject, attachment filename, content type supplied by the report generator, and SMS text. Snapshot tests can help review copy, but targeted assertions are less noisy when only whitespace changes. Keep attachment bytes out of logs.

Operationally, count decisions by channel and reason, transport attempts, accepted responses, and later delivery outcomes as separate signals. A provider accepting a request is not proof that a person received it. Correlate those records with an internal event identifier, while excluding message bodies, report contents, email addresses, and phone numbers from routine telemetry.

## What to measure before copying this choice

Measure the delay from opt-out receipt to durable suppression, the age of preference data at the final decision, duplicate attempts per event and channel, and the share of jobs suppressed after they were queued. Those numbers reveal whether the last-moment check is doing real work. Also track template-change lead time and adapter-specific code size; local ownership is a poor bargain if every copy edit still requires transport-specific changes.

The decision rule is narrow: own templates locally when consistent policy, testability, and transport portability matter more than a hosted editor. Keep a hosted template only when its collaboration workflow is genuinely more valuable and the application can still enforce preferences and suppression before invoking it. Either way, do not delegate consent.

For the support-report flow, success means the attachment and message stay coherent, an SMS can be declined without blocking a permitted email, and a late opt-out beats an early queue snapshot. Measure those properties under retries and delayed jobs before adopting the pattern.

## Further reading

- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- https://resend.com/docs/introduction
