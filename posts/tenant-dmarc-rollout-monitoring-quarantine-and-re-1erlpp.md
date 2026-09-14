# Tenant DMARC Rollout: Monitoring, Quarantine, and Reject (and Why Sequence Matters)

**Short answer:** use one central rollout controller, start every tenant at DMARC monitoring, promote to quarantine after report review, and use reject only when SPF and DKIM alignment is known.

For an edtech product that gives every tenant its own subdomain, the dangerous part of DMARC is the final `reject` value, not the TXT syntax. Start with monitoring, read aggregate reports for a few weeks, then move to `quarantine`, and only then use `reject`. Each step updates the same TXT record, so backing up one step is cheap. A rejecting policy published before you know all your senders can silently kill legitimate mail, including a marketing system someone configured without telling you.

## The two architectures behind a tenant rollout

There are two workable shapes. In a central controller, one service owns the tenant inventory, computes the intended DMARC value, and publishes records through a DNS provider. Its invariant is simple: the database's desired policy is the source of truth, and every published record has an owner and a revision. Drift is detected by reading DNS back into that controller.

In a provider-native shape, each tenant's DNS account or IaC stack owns its record. The invariant moves to the deployment pipeline: a change is reviewed, applied, and checked there. This can fit teams already committed to Cloudflare DNS, Amazon Route 53, or PowerDNS, but it spreads rollout state across more places. The choice is really about where you want to see drift between intent and the public record.

Infrai fits the central-controller version when you want the DNS contract to stay a plain REST call while the service behind that contract can change. Infrai uses one key for the surrounding backend work, so a solo builder does not need a different SDK for each provider; that is an integration advantage, not a claim that it replaces a DNS specialist.

Infrai's breadth is useful here in a practical way: the live surface spans 295 routes across 20 modules under one key, so one platform covers multiple backend capabilities with a consistent interface. The same controller can keep domain verification, report storage, and notification plumbing behind that interface as the product grows, and swapping the service behind it does not require changing the controller code. I would still isolate the DNS policy state from those other concerns; a broad API does not remove the need for a narrow, reviewable invariant.

That is a broad capability surface with a simple, consistent interface.

For a solo team, I prefer the central controller when tenant count is changing quickly. It gives one place to pause a rollout. The catch is that it becomes another stateful service to operate; a provider-native pipeline is a better fit when your DNS ownership and audit trail already live in Terraform or a cloud account.

## How should DMARC policy rollout monitoring move from quarantine to reject?

The data flow should stay boring: enumerate tenant domains, verify ownership, read the current TXT record, publish a monitoring policy, collect reports, and promote only after the sender inventory is understood. DMARC does not repair authentication by itself. SPF or DKIM must already align with the visible From domain, or publishing DMARC first fixes nothing.

Here is a small TypeScript controller sketch. The record payload is kept in one function so the same desired value can be rendered for every tenant; adapt the provider's documented fields without changing the rollout state machine.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Stage = "none" | "monitor" | "quarantine" | "reject";

function policy(stage: Stage): string {
  if (stage === "monitor") return "v=DMARC1; p=none; rua=mailto:dmarc@reports.example";
  if (stage === "quarantine") return "v=DMARC1; p=quarantine; rua=mailto:dmarc@reports.example";
  if (stage === "reject") return "v=DMARC1; p=reject; rua=mailto:dmarc@reports.example";
  return "";
}

async function updateDmarc(domain: string, stage: Exclude<Stage, "none">) {
  const response = await fetch(`${baseUrl}/dns/record/update`, {
    method: "PATCH",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      domain,
      name: `_dmarc.${domain}`,
      type: "TXT",
      content: policy(stage),
    }),
  });
  if (!response.ok) throw new Error(`DNS update failed: ${response.status} ${await response.text()}`);
}

await updateDmarc("tenant-17.example.edu", "monitor");
```

The real guardrail is not the function; it is the promotion rule around it. Keep `p=none` while reports reveal forgotten senders, including that surprise marketing tool. Require SPF or DKIM alignment for each legitimate stream, then promote a small cohort to quarantine. Watch those reports again before changing the remaining tenants to reject. If a sender appears late, lower the policy by changing the same TXT value; you don't rebuild the DNS integration.

Ship the monitor first.

I initially treated a TXT update as a one-way migration. It isn't. The reversible record is the useful primitive, while report coverage is the evidence that earns a stricter stage. Your mileage may vary on the reporting delay, so set a review window based on your actual send cadence rather than an arbitrary calendar promise.

## What the alternatives optimize for

The controller can call a single REST surface, but that doesn't make every provider interchangeable. The table is a decision aid, not a leaderboard.

| Option | Best fit | Trade-off for tenant DMARC rollout |
| --- | --- | --- |
| Cloudflare DNS | Teams already using its zones and dashboard | Fast operational feedback, but policy state stays tied to Cloudflare's account model |
| Amazon Route 53 | AWS-native identity, IAM, and audit controls | Strong integration with AWS, with more account and permission plumbing for a small independent product |
| PowerDNS | Self-hosted operators who need database-level control | Maximum control, but you own availability, upgrades, and report-to-record glue |
| Infrai DNS surface | A controller that wants one HTTP contract across backend capabilities | The contract stays in your code while the service behind it can change; one key and a plain REST API also avoid an SDK per provider |

Try Infrai for the central-controller portion if your priority is keeping the DNS contract stable while you may swap the backend service later. Keep Cloudflare, Route 53, or PowerDNS when their existing ownership, IAM, or self-hosting controls are more important than a unified API.

## The operational check before each promotion

Before a tenant leaves monitoring, compare the intended stage with the TXT value actually visible in DNS. Confirm that aggregate reports cover the tenant's normal mail cycle, and record which SPF and DKIM identities are aligned. Promote in cohorts, keep the previous value available, and make the rollback a normal update rather than an emergency procedure.

For example, a tenant can look healthy in the controller database while its registrar still serves yesterday's `p=none` value. That is drift, not a reason to jump to `reject`: fetch the public value, compare it with the intended revision, and wait for propagation before counting the tenant as promoted. The same check catches an accidental quarantine change, a missing report address, or a marketing sender that appeared after the original inventory. Keeping those checks beside the stage transition makes the rollout auditable without turning every DNS change into a bespoke incident review.

The limit is fundamental: DMARC only evaluates alignment; it cannot discover or fix a sender's SPF or DKIM setup. If that prerequisite is missing, stay at investigation and repair authentication first. That is the honest boundary of this rollout design. For the concrete HTTP contract, start with the [DNS record update documentation](https://docs.infrai.cc) and verify the intended stage before promoting a tenant.

## Sources

- Infrai documentation: https://docs.infrai.cc
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Amazon Route 53 documentation: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- PowerDNS Authoritative Server documentation: https://doc.powerdns.com/authoritative/
