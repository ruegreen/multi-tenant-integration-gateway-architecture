# Architecture Diagrams

Detailed diagrams for the multi-tenant integration gateway. All diagrams describe
design and control flow; none describe implementation internals.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Providers["Channel Providers"]
        TG["Telegram"]; LN["LINE"]; IG["Instagram"]; SL["Slack"]
        SM["SMS / MMS"]; RC["RCS"]; MT["Teams"]; WX["Enterprise Msg"]
    end

    subgraph GW["Integration Gateway"]
        subgraph Edge["Webhook Edge"]
            SHARED["Shared endpoints"]
            SCOPED["Tenant-scoped endpoints"]
            RESOLVE["Identity Resolution"]
        end
        ADAPT["Channel Adapters"]
        CORE["Inbound Processor"]
        DISPATCH["Outbound Dispatch"]
        REC["Rejection Recovery"]
        ATT["Attachment Store"]
        KEYS["Key Store"]
        TOK["Token Scheduler"]
        ADMIN["Admin UI"]
    end

    subgraph CC["Contact Center Platform"]
        TASK["Task lifecycle"]
        AGENT["Agent Desktop"]
    end

    DB[("Document Store")]

    Providers --> SHARED & SCOPED --> RESOLVE --> ADAPT --> CORE --> TASK
    TASK --> DISPATCH --> ADAPT --> Providers
    CORE <--> ATT
    TASK -.->|"rejects attachment"| REC --> CORE
    KEYS & TOK --> ADAPT & CORE
    ADMIN --> DB
    GW <--> DB
    AGENT --> ATT
```

---

## 2. The Record Hierarchy

Each record points at the one above it. This shape is what lets one deployment
serve many organizations.

```mermaid
flowchart TD
    TOK["<b>Platform token</b><br/>name: org-a-token<br/>client credentials + access/refresh<br/><i>refreshed in the background</i>"]
    T["<b>Tenant</b> = one organization<br/>slug: org-a<br/>platform token: <b>org-a-token</b> (by name)<br/>region · org id · webhook secret refs"]
    C1["<b>Channel</b>: messaging bot<br/>routing key = webhook secret token<br/>platform channel + business address"]
    C2["<b>Channel</b>: phone number<br/>routing key = E.164 number<br/>platform channel + business address"]
    C3["<b>Channel</b>: workspace bot<br/>routing key = app id<br/>platform channel + business address"]
    CONV["<b>Conversation</b> (one per open task)<br/>taskId ↔ tenant + channel + customer"]

    TOK -->|"referenced by name"| T
    T --> C1 & C2 & C3
    C1 -.->|"created on first message"| CONV
    C2 -.-> CONV
    C3 -.-> CONV
```

> **Channels hold no platform credentials.** Every call made for a channel uses
> its tenant's token. Set once per organization; every channel inherits,
> including ones added later.

---

## 3. Message Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant Cust as Customer
    participant Prov as Channel Provider
    participant GW as Gateway
    participant DB as Store
    participant CC as Contact Center Platform

    Cust->>Prov: sends a message
    Prov->>GW: POST /<channel>/inbound
    GW-->>Prov: 200 immediately
    GW->>GW: routing key → channel → tenant<br/>verify channel signature
    GW->>DB: open conversation for this customer?
    alt none open
        GW->>CC: Create Task (tenant's token)
        GW->>DB: store conversation (taskId ↔ tenant, channel, customer)
    else already open
        GW->>CC: Append message to taskId
    end
    CC-->>GW: lifecycle event (accepted / failed)
    Note over CC: flow or agent replies
    CC->>GW: POST /outbound (taskId, text, attachments)
    GW->>DB: taskId → conversation → tenant + channel
    GW->>GW: verify tenant's asset secret
    GW->>Prov: send with that channel's credentials
    Prov->>Cust: reply delivered
    CC-->>GW: lifecycle event (task ended)
    GW->>Prov: closing notice
    GW->>DB: delete conversation
```

---

## 4. Three Identity Resolutions

Each webhook class is resolved by whatever evidence its payload actually
carries. No scheme is forced onto a payload shape nobody controls.

```mermaid
flowchart TD
    IN["Webhook arrives"] --> WHICH{"Which class?"}

    WHICH -->|"channel inbound<br/>(customer message)"| RK["Extract the ROUTING KEY<br/>from the provider's payload"]
    RK --> RKU["Globally unique across all tenants<br/>(enforced at write time)"]
    RKU --> CH["→ exactly one channel → one tenant"]

    WHICH -->|"outbound<br/>(agent / flow reply)"| TID["Extract the TASK ID"]
    TID --> CONV["→ stored conversation → tenant"]
    CONV --> SIG1["Verify with that tenant's asset secret"]

    WHICH -->|"lifecycle events"| SIGV["Find the tenant whose<br/>SUBSCRIPTION SECRET verifies<br/>the signature"]
    SIGV --> FILT["Filter to that tenant's<br/>channels and addresses"]
    SIGV -.->|"identity AND authenticity<br/>in one operation"| FILT

    CH & SIG1 & FILT --> PROC["Process"]
```

### Routing keys by channel type

| Channel | Routing key | Arrives as |
|---|---|---|
| Messaging bot A | Webhook secret token | Request header |
| Messaging bot B | Bot user id | `destination` field |
| Social DM | Business account id | Entry object id |
| Workspace DM | Workspace/team id | `team_id` |
| SMS / MMS | Service number (E.164) | `serviceNumber` |
| RCS | Application id | `rcsAppID` |
| Enterprise chat | Bot application id | Recipient id |
| Platform messaging | Subscription id | Top-level `id` |

---

## 5. Dual Addressing

```mermaid
graph LR
    subgraph Shared["Shared form — one URL for every tenant"]
        S1["/api/v2/byoc/&lt;channel&gt;/inbound"]
    end
    subgraph Scoped["Tenant-scoped form"]
        S2["/api/v2/byoc/t/&lt;slug&gt;/&lt;channel&gt;/inbound"]
    end
    S1 --> R1["Tenant resolved from the request"]
    S2 --> R2["Only that tenant's traffic accepted;<br/>anything else fails loudly"]
    R1 & R2 --> P["Same processing"]
```

Both forms are live simultaneously; operators choose per tenant. Shared URLs are
safe *because* routing keys are globally unique.

---

## 6. Token Refresh Under a Lease

```mermaid
sequenceDiagram
    participant A as Instance A
    participant B as Instance B
    participant DB as Store
    participant P as OAuth Provider

    Note over A,B: both wake on the refresh schedule
    A->>DB: acquire lease on token X
    B->>DB: acquire lease on token X
    DB-->>A: granted
    DB-->>B: denied — already leased
    Note over B: skips; no concurrent refresh
    A->>P: refresh
    alt success
        P-->>A: new access + refresh token
        A->>DB: store, release lease
    else transient error (network / 5xx)
        P-->>A: failure
        A->>DB: record last error, release lease
        Note over A,DB: credential NOT invalidated
    else refresh token rejected
        P-->>A: invalid_grant
        A->>P: re-mint via org admin integration
        P-->>A: fresh token
        A->>DB: store, release lease
    end
```

> Concurrent refresh is the classic multi-instance failure: the second refresh
> presents a rotated-away token and the provider revokes the grant. It does not
> exist on one instance, and appears the day you scale out.

---

## 7. Credential Store

```mermaid
graph TB
    subgraph Store["Key Store — the only place values live"]
        VK["Vendor keys<br/>(gateway authenticates TO a vendor,<br/>or verifies its webhook)"]
        GK["Gateway API keys<br/>(callers authenticate TO the gateway)"]
    end

    subgraph Refs["Everything else holds references"]
        T["Tenants"]; C["Channels"]
    end

    T & C -->|"by name"| Store
    Store --> ENC["AES-256-GCM at rest"]
    ENC --> FP["Key fingerprint recorded"]
    FP --> CHK{"Fingerprint matches<br/>on startup?"}
    CHK -->|no| REFUSE["REFUSE to write secrets.<br/>Do not half-encrypt the store."]
    CHK -->|yes| OK["Normal operation"]
    Store --> USE["Usage index:<br/>an in-use key cannot be deleted"]
    Store --> AUD["Reveal is audit-logged;<br/>values never appear in<br/>responses or logs"]
    Store --> HASH["One channel secret is stored<br/>HASHED — verification needs<br/>only comparison"]
```

---

## 8. Rejection Recovery Ladder

```mermaid
flowchart TD
    SEND["Send message with attachment"] --> R{"Platform response"}
    R -->|accepted| DONE["Delivered"]
    R -->|"policy rejects content"| TYPE{"Is it an image?"}
    TYPE -->|yes| PDF["Wrap in a one-page PDF,<br/>resend as a real attachment"]
    PDF --> R2{"Accepted?"}
    R2 -->|yes| DONE2["Agent views it in Desktop"]
    R2 -->|no| LINK
    TYPE -->|no| LINK["Resend as text carrying a<br/>signed, expiring link"]

    R -->|"accepted, then scan<br/>DROPS the attachment"| FOLLOW["Follow up with the<br/>file's secure link"]
    R -->|"conversation already open"| STALE["End the stale task,<br/>recover automatically"]

    LINK & FOLLOW & DONE2 --> AGENT["Agent has the file"]
```

> The rejection is a property of the receiving organization's policy, not of the
> customer's file. Asking a customer to resend makes them solve a configuration
> problem they cannot see.

---

## 9. Attachment Handling

```mermaid
flowchart LR
    CUST["Customer sends a file"] --> DL["Gateway downloads it"]
    DL --> GFS[("Stored in GridFS")]
    GFS --> URL["Signed, expiring URL issued"]
    URL --> CC["Platform fetches it"]
    GFS --> SWEEP["Hourly sweep deletes files<br/>past the retention period;<br/>their links stop working"]
    GFS --> WIDGET["Agent Desktop widget lists a<br/>customer's files across this and<br/>earlier conversations"]
    DL --> ALLOW["Outbound fetches restricted<br/>by host allowlist (SSRF)"]
```
