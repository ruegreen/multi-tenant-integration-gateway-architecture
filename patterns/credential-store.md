# Credential Store Pattern

One encrypted store holds every secret value. Everything else holds a reference
to it by name.

---

## Both Directions in One Store

| Kind | Direction | Examples |
|---|---|---|
| **Vendor key** | The gateway authenticates *to* a vendor, or verifies a vendor's webhook | Bot tokens, channel secrets, service keys, platform webhook secrets |
| **Gateway API key** | A caller authenticates *to* the gateway | Machine-to-machine clients on the data endpoints |

Keeping both in one store is deliberate. They are the same operational object —
a secret with a lifetime, an owner and a rotation story — and splitting them by
direction produces two half-featured stores where one has an audit trail and
the other does not.

---

## Reference, Never Copy

```
Channel
  name
  routingKey
  vendorKeyRefs: { botToken: "org-a-bot", webhookSecret: "org-a-hook" }   # NAMES
```

```
sendOutbound(channel, message):
    token = keyStore.resolve(channel.vendorKeyRefs.botToken)   # looked up now
    provider.send(token, message)
```

Three properties follow, and none is available if values are copied:

1. **Changing a value takes effect everywhere at once.** One key shared by six
   channels is rotated once.
2. **The store knows where each key is used.** A usage index is maintainable
   only because the references are explicit.
3. **An in-use key cannot be deleted.** The store refuses, and names what is
   using it.

```
deleteKey(name):
    uses = usageIndex(name)
    if uses: refuse("in use by: " + uses.join(", "))
    ...
```

The indirection feels like overhead right up to the first rotation, at which
point it is the only reason the rotation is possible.

---

## Encryption at Rest, With a Fingerprint

```
encrypt(value):
    return { ciphertext: aes256gcm(value, KEY),
             keyFingerprint: fingerprint(KEY) }

startup():
    stored = anyStoredFingerprint()
    if stored and stored != fingerprint(KEY):
        enterReadOnlySecretsMode()
        log("encryption key mismatch — refusing to write secrets")
```

A gateway started with a different key **refuses to write secrets** rather than
proceeding.

### Why refusing beats proceeding

Starting with the wrong key is not exotic — it is a misconfigured environment
on a newly provisioned instance. Without the check, that instance cheerfully
encrypts *new* secrets with a key nothing else possesses, while being unable to
read the existing ones.

The result is a store where some records are readable and some are not, with no
record of which is which and no way to reconstruct the boundary after the fact.
Refusing to write keeps the damage at exactly zero and makes the
misconfiguration obvious on the first startup rather than on the first restore.

### Migration is automatic

Pre-existing plaintext records are encrypted on first start, so turning
encryption on is not a manual data operation somebody has to plan, schedule and
half-finish.

---

## Values Do Not Leave

```
GET /keys          -> [{ name, type, usedBy, enabled, updatedAt }]   # no values
GET /keys/:name    -> same shape
POST /keys/:name/reveal
                   -> { value }   + audit log entry (who, when, which key)
```

- Values never appear in API responses, logs or error messages by default.
- **Reveal is a distinct, audited operation** — by key name, so the audit trail
  is useful without itself containing secrets.
- Operators can enter their own value or have one generated.

---

## Hash Where Comparison Is Enough

One channel type authenticates its webhooks with a secret token the provider
echoes back in a header. Verification needs only equality, so the record stores
a **hash**:

```
verify(channel, request):
    presented = request.headers[SECRET_HEADER]
    return constantTimeEqual(hash(presented), channel.webhookSecretHash)
```

The channel record cannot leak a credential it does not hold. This is worth
doing anywhere a stored secret is only ever compared — the inability to read it
back is the feature.

---

## What Lives Elsewhere, and Why

OAuth tokens — platform service-app tokens and integration tokens — are
encrypted the same way but kept on their own records, because they carry
lifecycle state the key store has no concept of: expiry, refresh tokens, last
refresh error, lease ownership. See [token-lifecycle.md](token-lifecycle.md).

Putting them in the key store would mean either teaching it about OAuth or
storing lifecycle state somewhere other than the thing it describes.
