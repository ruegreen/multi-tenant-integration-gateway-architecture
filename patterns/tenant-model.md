# Tenant Model Pattern

Four record types in a strict hierarchy, and one write-time uniqueness rule that
removes an entire class of runtime ambiguity.

---

## The Hierarchy

```
Platform token   ← OAuth credentials for one organization's service app
      ↑ referenced by name
   Tenant        ← one organization: token name, region, org id, secret refs
      ↑ contains
   Channel       ← one endpoint: routing key, key refs, platform mapping
      ↓ creates
Conversation     ← one per open task: taskId ↔ tenant + channel + customer
```

Each record points at the one above it. Nothing skips a level.

---

## Credentials Live on the Tenant

```
callPlatform(channel, operation, payload):
    tenant = channel.tenant
    token  = tokenStore.get(tenant.platformTokenName)    # by NAME, always
    return platformApi(operation, payload, token)
```

A channel never stores a platform credential. It cannot — the field does not
exist on the record.

### Why not put the credential on the channel

It is the obvious shape, and it is the decision that makes rotation impossible.
An organization with nine channels has nine copies. Rotating means finding all
nine, and nothing detects the one that was missed until it fails in production,
on whichever channel happens to be used next.

Inheritance also means a channel added six months later is already configured.
There is no step where someone pastes the credential again.

**Trade-off:** a channel cannot use different credentials from its siblings. An
organization that genuinely needs two credential sets is modelled as two
tenants, which is the honest representation of what that is.

---

## Routing Keys Are Globally Unique

```
createChannel(tenant, type, routingKey, ...):
    existing = channels.findOne({ type, routingKey })    # across ALL tenants
    if existing:
        refuse("routing key already used by channel "
               + existing.name + " in tenant " + existing.tenant)
    ...
```

Enforced at **write time**, in the admin interface, across the entire
deployment — not per tenant.

### What this buys

Because the key is globally unique, an inbound webhook carrying it maps to
exactly one channel and therefore one tenant. There is nothing to disambiguate
at request time:

```
resolveInbound(channelType, payload):
    key     = extractRoutingKey(channelType, payload)
    channel = channels.findOne({ type: channelType, routingKey: key })
    if not channel: reject("unknown routing key")
    return channel, channel.tenant                       # no ambiguity, ever
```

This is what makes a **shared** webhook URL safe. Without the uniqueness rule,
a shared URL would need some other tenant hint — a query parameter in every
provider's configuration, fragile and silently wrong when mistyped.

### What it costs

Two tenants cannot share one bot, one phone number or one workspace. That is
correct rather than unfortunate: if they did, there would be no principled way
to decide whose conversation an inbound message belongs to. The constraint
surfaces a genuine modelling conflict at configuration time instead of
producing a coin-flip at runtime.

**The general rule:** when a system must later ask *"which of these does this
belong to?"*, check whether the answer could have been guaranteed at
configuration time. A configuration error a human sees immediately beats a
routing error discovered from a customer complaint.

---

## Conversations Are Ephemeral Join Records

```
Conversation
  taskId        the platform's identifier for the open interaction
  tenant        resolved once, at creation
  channel       which endpoint the customer used
  customer      provider-side identifier
```

Created when a task opens, deleted when it ends. They exist to answer one
question — *given a task id from an outbound webhook, whose is it?* — and are
the only record with a lifetime measured in minutes.

Keeping them separate from the channel record matters: a channel is
configuration, a conversation is state, and conflating the two would put
high-churn writes on the record that configuration reads from.

---

## What the Hierarchy Makes Easy

| Operation | Because |
|---|---|
| Add a channel | Its tenant already has credentials |
| Rotate a platform credential | One token record, referenced by name |
| Add an organization | A tenant plus its token; no deployment |
| Answer "what does this tenant have?" | One query, one level down |
| Prevent cross-tenant leakage | Resolution always terminates at exactly one tenant |
