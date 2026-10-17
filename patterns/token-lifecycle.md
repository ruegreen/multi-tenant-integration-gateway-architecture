# Token Lifecycle Pattern

OAuth refresh looks like a chore and is a distributed-systems problem. Three
rules keep it from taking an integration down.

---

## 1. Refresh Takes a Database Lease

```
refreshDue(token):
    lease = store.acquireLease(token.name, ttl = LEASE_TTL)
    if not lease:
        return                                  # another instance has it
    try:
        newCreds = provider.refresh(token.refreshToken)
        store.save(token.name, newCreds)
    finally:
        store.releaseLease(token.name, lease)
```

### Why this is not optional

Concurrent refresh against most OAuth providers **invalidates the grant**. The
second refresh presents a refresh token the provider has already rotated away
from, the provider treats that as a replay, and it revokes.

The defect has three properties that make it dangerous:

| Property | Consequence |
|---|---|
| Does not exist on one instance | Absent in development |
| Does not exist in a one-instance deployment | Absent in early production |
| Appears on the day of horizontal scaling | Arrives with no code change to blame |

And its symptom is a revoked grant requiring manual re-authorization, not an
error that retries away. A lease is cheap; discovering you needed one is not.

The lease has a TTL so a crashed instance cannot hold it forever.

## 2. A Transient Error Never Invalidates a Credential

```
except ProviderError as e:
    if e.isTransient():                 # network failure, timeout, 5xx
        store.recordLastError(token.name, e)    # surfaced in the UI
        return                                  # credential UNTOUCHED
    if e.isInvalidGrant():
        return handleInvalidGrant(token)
```

Treating a 503 as a revocation turns a provider's bad minute into an outage
that needs a human to re-authorize — at exactly the moment the provider is
already struggling and the re-authorization is most likely to fail too.

**Every integration that stores a credential needs an explicit policy for which
errors invalidate it, and the default should be none of them.** Only an
unambiguous rejection of the grant itself counts.

The last error is recorded and shown in the interface, so a failing refresh is
visible without being destructive.

## 3. A Dead Refresh Token Re-Mints Itself

```
handleInvalidGrant(token):
    if token.mintedVia == ORG_ADMIN_INTEGRATION:
        fresh = provider.mint(token.appId, scope = APP_TOKEN_SCOPE)
        store.save(token.name, fresh)           # self-healed
    else:
        store.markNeedsReauthorization(token.name)
```

Tokens minted through an organization-admin integration can be re-minted
without a human, so the common expiry path heals itself instead of paging
someone. Tokens that genuinely require an interactive authorization are marked
as needing it, and say so in the interface.

---

## Refresh Runs Ahead of Expiry

```
scheduler():
    for token in store.all():
        if token.expiresAt - now() < REFRESH_WINDOW:
            refreshDue(token)
```

Background refresh on a window, rather than refresh-on-401. Refreshing lazily
means the first request after expiry pays the latency and, worse, that a
provider outage is discovered by a customer-facing request rather than by a
scheduler with no one waiting on it.

---

## Minting Paths

Several ways a token can originate, all converging on the same record shape:

| Origin | How |
|---|---|
| Minted via an org-admin integration | Gateway calls the provider's app-token endpoint |
| Pasted from a developer portal | Operator supplies access and refresh tokens |
| JWT assertion | Signed assertion exchanged for a token |
| Authorization code | Popup flow; the gateway's callback completes the exchange |

A fallback that accepts a pasted redirect URL covers providers that cannot
redirect to the gateway. Keeping all four on one record shape is what lets a
single scheduler refresh all of them.

---

## What the Interface Must Show

A token subsystem that works invisibly is indistinguishable from one that
stopped working, so the state is surfaced:

- when each token was last refreshed, and when it next will be
- the last error, if any, without clearing the credential
- which tokens need interactive re-authorization
- which tenant references each token by name

Manual refresh and manual re-mint are available as explicit actions, because
the first thing an operator wants during an incident is to try the thing the
scheduler is about to try anyway.
