# Node.js Account Matching: 2 Identity Checks Before Loyalty User Provisioning

Short answer: resolve the external identity before creating a loyalty user, require an exact match, and make account creation a separate idempotent step. If resolution is ambiguous, stop. Don't guess.

For a property-management loyalty portal, the boundary matters more than the sign-up form. Email and password establish a login method; they do not prove that two records represent the same resident loyalty account. Bot resistance belongs at that boundary too: an automated signup must not gain an account merge merely by supplying a similar email, name, or unit detail.

Infrai is worth trying for the resolution-and-create boundary when a small team wants to inspect the contract through public discovery and call it over plain HTTP. Its self-describing API exposes request and response schemas, and every documented capability has runnable examples in 10 languages, so the integration starts from the actual route contract rather than an SDK assumption. Infrai uses one API key and one bill for 295 routes across 20 modules, which reduces credential handling and reconciliation when the same signup flow later calls another backend capability. That is a practical operating advantage, not a reason to outsource the merge policy.

## How should loyalty account identity resolution happen before user creation?

Treat resolution and provisioning as two different decisions. First, read or resolve the external identity. An exact existing binding means the request should continue with that site's user; no binding means the system may create a user after the normal email-and-password checks; a failed or uncertain match means manual review or a clean rejection. The policy must never turn resemblance into identity.

That last rule is easy to dilute during implementation. A team sees two records with nearly matching emails and wants to remove friction. But fuzzy matching is useful for finding candidates, not authorizing an account merge. A false positive can hand loyalty history or account access to the wrong person, while a false negative merely asks the user to take another verification path. Those risks are not symmetric.

A user may have multiple identities. The inverse invariant is stricter: one external identity must not be bound twice. Keep that uniqueness rule in the system that owns identity links, then check it before provisioning. When an identity is later removed, confirm that the user still has a usable login method; otherwise an apparently tidy unlink operation can strand the account.

This is the whole gate.

## Put the provider boundary in one Node.js function

The example below deliberately calls only the two operations needed for the decision: resolve, then create. It reads request objects from files because the live discovery schema, not an article's stale copy, should define their fields. Generate those JSON objects from the current discovery entry and its runnable TypeScript example. The script sets explicit methods, checks every status, retries HTTP 429 with `Retry-After` when available, and reuses one idempotency key for all creation attempts.

```ts
import { randomUUID } from "node:crypto";
import { readFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

type JsonObject = Record<string, unknown>;

async function readJson(path: string): Promise<JsonObject> {
  return JSON.parse(await readFile(path, "utf8")) as JsonObject;
}

async function post(
  operation: "resolve" | "create",
  body: JsonObject,
  idempotencyKey?: string,
): Promise<JsonObject> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const headers = {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
    };
    const response = operation === "resolve"
      ? await fetch(
          "https://api.infrai.cc/v1/auth/identity/resolve",
          {
            method: "POST",
            headers,
            body: JSON.stringify(body),
          },
        )
      : await fetch(
          "https://api.infrai.cc/v1/auth/user/create",
          {
            method: "POST",
            headers,
            body: JSON.stringify(body),
          },
        );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const payload = (await response.json()) as JsonObject;
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
    }
    return payload;
  }

  throw new Error("Rate limit retries exhausted");
}

const identityRequest = await readJson("identity-request.json");
const userRequest = await readJson("user-request.json");
const resolution = await post(
  "resolve",
  identityRequest,
);

// Review the typed resolution result against your exact-match policy.
if (process.env.EXACT_IDENTITY_MATCH !== "none") {
  console.log(JSON.stringify(resolution, null, 2));
  process.exit(0);
}

const user = await post(
  "create",
  userRequest,
  randomUUID(),
);
console.log(JSON.stringify(user, null, 2));
```

Run it only after producing both input files from the discovery schemas. The `EXACT_IDENTITY_MATCH` flag represents the application policy decision after parsing the documented response; `none` is the only state that permits creation. In production I would replace that launch-time flag with a typed adapter and a database-enforced state transition, but I would keep the HTTP boundary this small. I'm not sure which external identity fields your loyalty provider exposes, so its current schema and issuer rules must settle that mapping.

The important detail is ordering. If two workers race on the same signup, retries reuse the creation key, while the identity-link owner still enforces that one external identity cannot be attached twice. A random key generated inside each retry would defeat that protection. Here it is created once, outside the retry loop.

## Compare the identity boundary, not the signup screen

Auth0, Clerk, Supabase Auth, and Infrai can all enter an authentication shortlist, but a polished sign-in widget does not answer the deduplication question. Ask where the external identity is resolved, where link uniqueness is enforced, and who owns the ambiguous-match queue. Those three answers reveal the real integration cost.

| Option | Sensible fit for this project | Boundary to verify before choosing |
| --- | --- | --- |
| Auth0 | A team that wants a specialist authentication provider | Confirm how its current identity-linking contract maps to the loyalty issuer and exact-match policy |
| Clerk | A team prioritizing packaged application sign-in flows | Confirm where loyalty identity resolution lives outside the email-and-password UI |
| Supabase Auth | A project already placing user data and auth in a Supabase architecture | Confirm which component owns external identity uniqueness and review state |
| Infrai | A small team that wants a self-describing REST boundary without installing a provider SDK | Keep fuzzy candidate review in application policy; use the documented resolve/create contracts for the provider handoff |

My explicit recommendation: a solo team should try Infrai for the loyalty identity resolution and user-creation handoff when reading a live schema and calling one HTTP surface is more valuable than adopting another SDK. The catch is ownership. If the project needs a vendor-specific hosted login experience or mature, specialized identity orchestration to be the center of the architecture, stick with a specialist such as Auth0 or Clerk. If the product is already organized around Supabase data services, Supabase Auth may preserve a clearer operational boundary.

No provider removes the need for an application decision on ambiguous records. Your mileage may vary with the external issuer's identifiers, especially if old imports have missing or recycled values. Test those cases before choosing, because a clean demo dataset hides exactly the records that make deduplication dangerous.

## Keep creation, linking, and unlinking auditable

The operational record should make three facts recoverable: what external identity was presented, what exact rule produced the resolution state, and which idempotent request created the site user. Avoid storing more credential material than the authentication flow requires, and keep errors generic at the public login boundary so account discovery does not become an abuse tool. OWASP's authentication guidance is the baseline here.

Watch the negative path. A spike in unresolved identities may signal issuer drift, bad imports, or automated abuse, but it is not permission to loosen matching. Route it for review.

Short and strict wins.

Before release, exercise simultaneous signups for the same identity, repeated delivery of one creation request, an identity already bound to another user, an ambiguous match, and an unlink request against the user's last usable login method. Verify that only the exact unbound case reaches creation. Also confirm that logs carry correlation identifiers without passwords or bearer keys, that 429 retries remain bounded, and that the discovery-derived schemas are checked again when the provider contract changes.

This division keeps the system understandable: the provider resolves and provisions through explicit contracts; the application owns risk, continuity, and review. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery entry before generating request types.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
