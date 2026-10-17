# Channel Adapter Pattern

Eight providers, eight shapes. One adapter module each, and a deliberate refusal
to hide the differences behind a uniform interface that would misrepresent them.

---

## The Boundary

```
adapters/
  channelTypes.js      field definitions per channel type
  telegram.js  line.js  instagram.js  slack.js
  smsmms.js    rcs.js   teams.js      entmsg.js
inboundProcessor.js    shared: append-or-create, /end handling
dispatch.js            shared: outbound routing
verify.js              shared: signature primitives
```

Each adapter owns exactly four things:

| Responsibility | Why it is per-adapter |
|---|---|
| Extract the **routing key** | Every provider puts it somewhere different |
| **Verify** the inbound signature | Header name, algorithm and encoding all differ |
| **Normalize** inbound to the common message shape | Payload structures share nothing |
| **Render and send** outbound | Attachment models and size rules differ |

Everything else is shared. Adding a ninth channel is one module plus a field
definition — the contact center platform never changes, it just gains another
door.

---

## The Common Shape Is Deliberately Small

```
InboundMessage
  customerRef      provider-side identifier
  displayName      what the provider gives us, if anything
  text             may be empty when the message is attachment-only
  attachments[]    { filename, mimeType, bytes }
  rawRef           provider payload id, for troubleshooting
```

Small because every field added to a common shape is a field some adapter has
to fake. The temptation is a rich normalized model with avatar, profile,
read-receipt, typing state — and then six of eight adapters synthesize most of
it.

A narrow shape makes the gaps visible instead of inventing values. Where a
provider gives no real customer name, the adapter says so with a generic label
rather than fabricating one from an identifier.

---

## Capability Is Published, Not Abstracted

The uniform-interface instinct says: expose `sendAttachment` on every channel
and let each adapter do its best. That produces an API that lies — the call
succeeds and the customer receives something different per channel, with
nothing saying so.

Instead the differences are documented per channel and surfaced to operators:

| Difference | Reality |
|---|---|
| Reply windows | One platform restricts replies outside a 24-hour window |
| Prerequisites | One requires the customer to add the account before messaging |
| Directory scope | One is restricted to a single corporate directory |
| Message segmentation | SMS is split by the carrier, not by the gateway |
| Outbound files | Several support images only; other types fall back to a link |
| Format conversion | One inbound path converts a photo format the platform rejects |

That last row is instructive: the conversion happens on **one** channel because
that is the only one where the problematic format actually arrives. Applying it
everywhere would be speculative work defending against a case that does not
occur, and would hide the fact that the same format on another channel *will*
be rejected — which operators need to know.

---

## Shared Core: Append or Create

```
processInbound(channel, tenant, message):
    if message.text == END_COMMAND:
        return endTask(channel, tenant, message.customerRef)

    conv = conversations.findOne({ channel, customer: message.customerRef })
    if conv:
        return platform.appendMessage(conv.taskId, message, tenant.token)

    task = platform.createTask(channel, message, tenant.token)
    conversations.insert({ taskId: task.id, tenant, channel,
                           customer: message.customerRef })
```

This module is identical across every channel and was carried over unchanged
from the single-tenant predecessor. It is the part that genuinely *is* the same
everywhere, which is why it is the only large piece of shared behaviour.

---

## Where Adapters Must Not Reach

- **No adapter holds platform credentials.** It receives the tenant's token.
- **No adapter resolves tenancy.** Resolution happens at the edge; the adapter
  is handed its channel and tenant.
- **No adapter writes conversation state.** The shared processor owns it.
- **No adapter reads another adapter.** Shared behaviour moves to the core or
  stays duplicated; cross-imports between channels are how one provider's quirk
  becomes everyone's bug.

The test for whether something belongs in an adapter: *would this still be true
if the provider changed its API tomorrow?* If yes, it is core.
