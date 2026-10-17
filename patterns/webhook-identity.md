# Webhook Identity Pattern

Every entry point receives a payload shaped by someone else and must answer
*"whose traffic is this?"* from whatever evidence that payload happens to carry.
Three classes of webhook carry three different kinds of evidence, so there are
three resolutions.

---

## Why Not One Scheme

The tidy design is a tenant identifier on every request. It cannot be built
here: these are third-party payload shapes nobody controls, and none of them
carries one.

Forcing the scheme means putting a tenant hint in the URL or a query parameter
in every provider's configuration — fragile, invisible once configured, and
silently wrong when mistyped. The alternative taken is to resolve from evidence
each payload genuinely contains.

---

## 1. Channel Inbound — Resolve by Routing Key

```
resolveInbound(channelType, request):
    key     = extractRoutingKey(channelType, request)   # per-adapter
    channel = channels.findOne({ type: channelType, routingKey: key })
    if not channel: reject(404)
    verifySignature(channel, request)                   # channel's own secret
    return channel, channel.tenant
```

Routing keys are globally unique per channel type, enforced at write time, so
the lookup is exact. See [tenant-model.md](tenant-model.md).

Each adapter knows where its provider puts the identifier — a header, a
`destination` field, an entry id, a service number — and that knowledge lives in
the adapter rather than in the resolver.

## 2. Outbound — Resolve by Task, Then Verify

```
resolveOutbound(request):
    taskId = request.body.taskId
    conv   = conversations.findOne({ taskId })
    if not conv: reject("no open conversation for this task")
    tenant = conv.tenant
    verifySignature(tenant.assetWebhookSecret, request)
    return conv.channel, tenant
```

The reply direction has no routing key — it originates from the platform, not
the provider. The task id is the only durable handle, and the conversation
record is what turns it into a tenant.

## 3. Lifecycle Events — Resolve *By* Verifying

```
resolveEvent(request):
    for tenant in tenantsWithEventSubscriptions():
        if verifySignature(tenant.eventsSecret, request):   # constant-time
            return tenant, filterToTenantChannels(tenant, request)
    reject("no tenant signature matched")
```

This is the interesting one. Identity and authenticity resolve in the **same
operation**: a payload that verifies against a tenant's secret both *is* that
tenant's and is *proven* to be. There is no separate "now check it's really
them" step to forget.

The cost is a loop over tenants with subscriptions. Bounded by tenant count,
and each check is a cheap HMAC — acceptable, and stated rather than hidden.

---

## Dual Addressing

Every webhook exists at two paths at once:

| Form | Shape | Behaviour |
|---|---|---|
| **Shared** | `/<channel>/inbound` | Tenant resolved from the request |
| **Tenant-scoped** | `/t/<slug>/<channel>/inbound` | Only that tenant's traffic accepted |

```
handleScoped(slug, channelType, request):
    channel, tenant = resolveInbound(channelType, request)
    if tenant.slug != slug:
        reject("traffic does not belong to this tenant")   # loud, not quiet
    return process(channel, tenant, request)
```

Both are live simultaneously and operators choose per tenant. Shared is simpler
and is correct because of the uniqueness rule. Scoped makes a misconfiguration
fail loudly rather than route oddly, which some organizations require.

Supporting both costs one routing prefix and one comparison.

---

## Acknowledge Immediately, Process Afterwards

```
webhookHandler(request, response):
    response.status(200).send()        # first, always
    enqueue(() => process(request))    # work happens after
```

Providers retry on timeout, and a retry is a duplicate message delivered to a
live agent. Returning immediately means slow downstream work can never
manufacture duplicates.

**The honest limit:** this prevents the gateway from *causing* retries. It does
not make the gateway idempotent against a provider's own retries or duplicate
platform events. Deduplication is a known gap, listed as one rather than
implied away by this pattern.

---

## Signature Verification Rules

- **Constant-time comparison** everywhere. A signature check that short-circuits
  on first mismatch leaks the signature a byte at a time.
- **Fail closed.** An unverifiable payload is rejected; there is no "process it
  anyway and log a warning" branch.
- **Hash where comparison is all that's needed.** One channel type's secret is
  stored as a hash, so the channel record cannot leak a credential it does not
  hold.
- **Replay protection** on the paths whose providers supply a timestamp.
