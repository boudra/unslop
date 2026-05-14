# Bolt-on Slop

Agents add new layers next to existing code instead of reshaping what's there. They see a problem and slap coordination on top, add a flag to route around the shape, or introduce a helper that duplicates existing semantics — rather than digging into the existing code and fitting the change into the shape that's already there.

## Signs

- Large additive diff for a small behavior change
- A new coordinator, manager, or orchestrator wrapping something that already coordinates
- A new flag, field, or parameter that lets one caller bypass the normal path
- A new helper whose semantics overlap an existing selector, reducer, action, hook, or effect
- Tests that assert internal calls (`toHaveBeenCalledWith`, spy counts) instead of observable behavior
- Code written *next to* existing code rather than modifying it

## Rule

Would this change look like a thoughtful edit to existing code, or a new layer next to it? If it's a new layer, stop. Re-read what exists, find the shape the change should fit into, and reshape instead.

## Conditional Accretion

A specific flavor of bolt-on: same discriminator (`plan`, `provider`, `kind`, `status`) branched in five different files, and each new case adds another branch in each file. Each local edit looks small. The cumulative shape is a maze.

```typescript
// Bad: plan logic scattered, every new plan = N edits
// api/upload.ts
if (user.plan === "free") maxFiles = 3;
else if (user.plan === "pro") maxFiles = 100;
else if (user.plan === "enterprise") maxFiles = 1000;

// ui/sidebar.tsx
const canUseBulkExport =
  user.plan === "pro" || user.plan === "enterprise" || user.plan === "team";

// billing/invoice.ts
if (user.plan === "team") applySeatPricing();
else if (user.plan === "enterprise") applyContractPricing();

// jobs/report.ts
const priority = user.plan === "enterprise" ? "high" : "normal";
```

```typescript
// Good: centralized policy, one place to add a plan
type Plan = "free" | "pro" | "team" | "enterprise";

interface PlanPolicy {
  maxFiles: number;
  canUseBulkExport: boolean;
  pricingModel: "flat" | "seat" | "contract";
  reportPriority: "normal" | "high";
}

const PLAN_POLICY: Record<Plan, PlanPolicy> = {
  free:       { maxFiles: 3,    canUseBulkExport: false, pricingModel: "flat",     reportPriority: "normal" },
  pro:        { maxFiles: 100,  canUseBulkExport: true,  pricingModel: "flat",     reportPriority: "normal" },
  team:       { maxFiles: 300,  canUseBulkExport: true,  pricingModel: "seat",     reportPriority: "normal" },
  enterprise: { maxFiles: 1000, canUseBulkExport: true,  pricingModel: "contract", reportPriority: "high"   },
};

function policyFor(plan: Plan): PlanPolicy {
  return PLAN_POLICY[plan];
}
```

**Widening pass.** When you spot a discriminator branched in three or more places, widen scope before patching. Trace where decisions for that discriminator currently live. Count the layers. If the same `if (kind === ...)` appears across modules, the next case shouldn't add another branch — it should centralize the policy first.

**Rule:** A new case in a discriminator-branched system is a structural signal, not a one-line change. Centralize, then add the case once.

**Smell:** Diff that adds the same `else if (status === "new-thing")` to four files. Boolean accumulation around the same enum (`isEnterprise`, `isTeam`, `isPaid`). Comments like "if you add a plan, also update X, Y, and Z."

## Single-Adapter Ports

Defining a port (interface) at a seam, with only one thing on the other side. The seam is hypothetical — nothing actually varies across it.

```typescript
// Bad: a port with one adapter, used by one caller, with no test substitute
interface NotificationSender {
  send(userId: string, body: string): Promise<void>;
}

class SendgridNotificationSender implements NotificationSender {
  async send(userId: string, body: string) { ... }
}

// Caller takes the port:
function notifyUser(sender: NotificationSender, userId: string, body: string) {
  return sender.send(userId, body);
}
// Production wires Sendgrid. No other adapter exists. The interface is indirection.
```

```typescript
// Good: until two adapters are justified, the caller depends on the concrete thing
async function notifyUser(userId: string, body: string) {
  return sendgrid.send(userId, body);
}

// When the test wants to substitute, *that's* the second adapter — and *now* the seam earns its keep.
```

**Rule:** **One adapter means a hypothetical seam. Two adapters means a real seam.** Don't introduce a port unless something actually varies across it (typically: production + test substitute, or two production implementations). A single-adapter port is just indirection; delete the interface and depend on the concrete thing until variance shows up.

**Smell:** An `interface` whose only `implements` is one class with the same surface. A constructor parameter typed as a port that's only ever passed one value. Tests that don't use the port — they call the concrete thing directly.
