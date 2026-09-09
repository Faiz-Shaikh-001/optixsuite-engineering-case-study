# Offline-First Synchronization

OptixSuite uses an offline-first architecture.

The central idea is simple:

> Normal application operations should succeed against the local database without requiring an active connection to the cloud.

Synchronization then propagates changes between devices and the server.

The difficult part is not sending HTTP requests.

The difficult part is maintaining a coherent distributed state when:

* devices can go offline;
* multiple devices can edit data;
* synchronization can fail halfway through;
* records can be deleted;
* old application versions may exist;
* data belongs to different stores;
* encrypted values must survive synchronization correctly.

---

# 1. Local Database as the Immediate Source

Application screens interact with local persistence.

```mermaid
flowchart LR
    USER[User]
        --> APP[Flutter Application]

    APP <--> LOCAL[Local Database]

    LOCAL <--> SYNC[Sync Engine]

    SYNC <--> CLOUD[Cloud Database]
```

This means normal operations do not depend directly on cloud availability.

For example, when creating a customer:

```mermaid
sequenceDiagram
    participant U as User
    participant A as Application
    participant D as Local Database
    participant S as Sync Engine
    participant C as Cloud

    U->>A: Create customer
    A->>D: Save customer locally
    D-->>A: Local write succeeds
    A-->>U: Customer appears immediately

    S->>D: Detect unsynchronized record
    S->>C: Push record
    C-->>S: Synchronization result
    S->>D: Update sync metadata
```

The user does not have to wait for the cloud round trip.

---

# 2. Why Offline-First Matters

Optical stores should remain operational during temporary infrastructure failures.

Possible scenarios include:

* unstable Wi-Fi;
* mobile hotspot usage;
* ISP downtime;
* high latency;
* temporary Supabase unavailability;
* device reconnection after extended offline use.

A cloud-dependent application could turn these events into business interruptions.

Offline-first moves that dependency away from the normal user interaction path.

---

# 3. The Cost of Offline-First

Offline-first does not eliminate complexity.

It relocates it.

```mermaid
flowchart TD
    OFFLINE[Offline-First Architecture]

    OFFLINE --> FAST[Fast Local Interaction]
    OFFLINE --> AVAILABLE[Network Independence]

    OFFLINE --> DISTRIBUTED[Distributed State]

    DISTRIBUTED --> CONFLICTS[Possible Conflicts]
    DISTRIBUTED --> RETRIES[Retry Logic]
    DISTRIBUTED --> VERSIONS[Version Tracking]
    DISTRIBUTED --> DELETIONS[Deletion Handling]
    DISTRIBUTED --> RECOVERY[Recovery Logic]
```

The user experience becomes simpler.

The synchronization system becomes harder.

---

# 4. Record Identity

Synchronization requires a stable identity that exists independently of a device's internal database identifier.

A local database ID is not sufficient because two devices may assign different internal IDs to the same logical record.

Conceptually, synchronized records therefore require a globally stable identifier.

```text
syncUuid
```

or an equivalent globally unique key.

Conceptually:

```mermaid
flowchart TD
    CLOUD["Customer<br/>syncUuid = abc-123"]

    CLOUD --> DEVICEA["Device A<br/>localId = 17<br/>syncUuid = abc-123"]

    CLOUD --> DEVICEB["Device B<br/>localId = 204<br/>syncUuid = abc-123"]
```

The device-specific IDs may differ.

The synchronization identity must not.

---

# 5. Synchronization Metadata

A synchronized record needs enough metadata to reason about its state.

A simplified conceptual model could include:

```text
syncUuid
storeId
version
updatedAt
deleted
syncStatus
```

The actual production representation can differ, but the system needs equivalent information.

Each field answers a different question.

| Metadata       | Purpose                                    |
| -------------- | ------------------------------------------ |
| `syncUuid`     | Which logical record is this?              |
| `storeId`      | Which store owns the record?               |
| `version`      | Which state is newer?                      |
| `updatedAt`    | When was the record modified?              |
| deletion state | Has the record been intentionally removed? |
| sync state     | Does the device have unsynchronized work?  |

---

# 6. Local Write Flow

A local write should complete independently of the network.

Conceptually:

```mermaid
flowchart TD
    A[User Updates Record]
        --> B[Validate Input]

    B --> C[Write Local Transaction]

    C --> D[Increment / Update Version Metadata]

    D --> E[Mark Record as Requiring Synchronization]

    E --> F[Commit]

    F --> G[Update UI]

    F --> H[Sync Engine Can Process Later]
```

The important property is:

**the network is not part of the local transaction.**

---

# 7. Push Synchronization

Push synchronization identifies local changes that have not yet been propagated to the cloud.

```mermaid
flowchart TD
    A[Start Push Cycle]
        --> B[Find Locally Changed Records]

    B --> C{Any records?}

    C -->|No| Z[Push Complete]

    C -->|Yes| D[Prepare Payload]

    D --> E[Apply Encryption / Encoding Requirements]

    E --> F[Send to Cloud]

    F --> G{Accepted?}

    G -->|Yes| H[Update Local Sync Metadata]
    G -->|No| I[Preserve Local Pending State]

    I --> J[Retry Later]
```

A failed network operation should not cause the local change to disappear.

The unsynchronized state remains until the system successfully reconciles it.

---

# 8. Pull Synchronization

Pull synchronization retrieves remote changes made by other devices.

```mermaid
flowchart TD
    A[Start Pull Cycle]
        --> B[Read Synchronization Watermark]

    B --> C[Request Changes After Watermark]

    C --> D[Receive Remote Records]

    D --> E[Validate Payload]

    E --> F[Decode / Decrypt]

    F --> G[Compare Local and Remote State]

    G --> H[Apply Valid Changes Transactionally]

    H --> I[Advance Watermark]

    I --> J[Pull Complete]
```

The watermark must only advance when the corresponding remote changes have been handled safely.

Otherwise records could be skipped permanently.

---

# 9. Synchronization Watermarks

A synchronization watermark represents the last safely processed point in the remote change stream.

Conceptually:

```mermaid
timeline
    title Remote Change Stream

    T1 : Record A updated
    T2 : Record B created
    T3 : Record C updated
    T4 : Record D deleted
```

If a device has safely processed through `T2`, its watermark should represent that position.

The next pull should begin after that point.

The important property is:

> The watermark represents successfully processed state, not merely downloaded state.

---

# 10. Why Watermark Ordering Matters

Consider this failure:

```mermaid
sequenceDiagram
    participant S as Sync Engine
    participant C as Cloud
    participant D as Local DB

    S->>C: Request changes after watermark 100
    C-->>S: Changes 101-120

    S->>S: Advance watermark to 120

    S->>D: Apply changes
    D-->>S: Failure while applying change 108
```

If the watermark was already persisted as `120`, the next synchronization could skip changes `108-120`.

The safer ordering is:

```mermaid
sequenceDiagram
    participant S as Sync Engine
    participant C as Cloud
    participant D as Local DB

    S->>C: Request changes after watermark 100
    C-->>S: Changes 101-120

    S->>D: Apply validated changes
    D-->>S: Transaction succeeds

    S->>D: Persist watermark 120
```

Only confirmed progress should advance synchronization state.

---

# 11. Version-Based Reconciliation

Once multiple devices can modify the same record, the system requires a way to determine which representation is newer.

The simplified model is:

```mermaid
flowchart TD
    A[Local Record + Remote Record]
        --> B{Compare Versions}

    B -->|Remote newer| C[Apply Remote State]
    B -->|Local newer| D[Preserve / Push Local State]
    B -->|Equal| E[No Change Required]
```

This prevents blindly overwriting state based only on whichever request arrived last.

---

# 12. Why Pure Last-Write-Wins Is Risky

A simple last-write-wins strategy based only on timestamps appears attractive.

However, distributed timestamps can be unreliable because:

* device clocks can differ;
* clocks can be manually changed;
* offline devices may reconnect much later;
* network ordering does not necessarily represent edit ordering.

Version-based state makes the system less dependent on client clock accuracy.

Timestamps can still be useful, but they should not automatically become the only conflict-resolution mechanism.

---

# 13. Conflict Scenarios

Consider:

```mermaid
sequenceDiagram
    participant A as Device A
    participant C as Cloud
    participant B as Device B

    C-->>A: Customer version 5
    C-->>B: Customer version 5

    Note over A: Device goes offline

    B->>C: Update customer -> version 6

    Note over A: Offline user also edits version 5

    A->>C: Reconnect and attempt push
```

Now two valid edits exist.

The system must determine whether to:

* reject the stale write;
* merge changes;
* preserve both for manual resolution;
* apply a domain-specific rule.

Not all record types necessarily require the same strategy.

Conflict resolution is a business-domain concern as much as a technical one.

---

# 14. Deletes Require Tombstones

Hard-deleting a synchronized record locally can create ambiguity.

Suppose Device A deletes a customer while Device B is offline.

If the cloud simply removes the row, Device B may later reconnect with an older local copy and recreate it.

A common solution is a tombstone or logical deletion marker.

```mermaid
flowchart LR
    RECORD[Active Record]
        --> DELETE[User Deletes]

    DELETE --> TOMBSTONE["Deletion Marker<br/>deleted = true"]

    TOMBSTONE --> SYNC[Propagate Deletion]

    SYNC --> DEVICES[Other Devices Remove / Hide Record]
```

The deletion itself becomes synchronized information.

---

# 15. Multi-Store Isolation

Every synchronization operation must respect store ownership.

It is not enough for the UI to filter records.

```mermaid
flowchart TD
    OWNER[Owner]

    OWNER --> STOREA[Store A]
    OWNER --> STOREB[Store B]

    STOREA --> DATAA[Store A Records]
    STOREB --> DATAB[Store B Records]

    DATAA --> SYNCA[Store A Sync Scope]
    DATAB --> SYNCB[Store B Sync Scope]
```

Store identity therefore needs to propagate through:

* record creation;
* local queries;
* synchronization payloads;
* server authorization;
* recovery;
* conflict handling.

This reduces the chance that one store accidentally receives another store's data.

---

# 16. Transaction Boundaries

Applying remote records individually can leave the database partially updated if synchronization fails halfway through.

For related changes, transactional application is preferable.

```mermaid
flowchart TD
    A[Receive Remote Batch]
        --> B[Validate Batch]

    B --> C[Begin Local Transaction]

    C --> D[Apply Record 1]
    D --> E[Apply Record 2]
    E --> F[Apply Record 3]

    F --> G{All Successful?}

    G -->|Yes| H[Commit]
    G -->|No| I[Rollback]
```

The exact transaction scope depends on record dependencies and batch size.

Large transactions improve atomicity but may increase contention and recovery cost.

---

# 17. Idempotency

Synchronization operations should be safe to retry.

A network failure may occur after the server accepted a request but before the client received the response.

```mermaid
sequenceDiagram
    participant D as Device
    participant C as Cloud

    D->>C: Push record version 8
    C->>C: Save version 8

    Note over D,C: Connection drops before response

    D->>C: Retry version 8
```

The second request should not produce duplicate logical data.

Stable record identity and version checks help make retries idempotent.

---

# 18. Encryption and Synchronization

Sensitive fields should not become plaintext merely because data is synchronized.

Conceptually:

```mermaid
flowchart LR
    DOMAIN[Domain Model]
        --> CODEC[Sync Payload Codec]

    CODEC --> ENC[Encrypt Sensitive Fields]

    ENC --> CLOUD[Cloud Payload]

    CLOUD --> DEC[Decrypt / Validate]

    DEC --> LOCAL[Local Persistence]
```

Cryptographic processing must occur at controlled boundaries.

The synchronization engine should not require feature-specific knowledge about which fields are sensitive.

---

# 19. Failure Handling

Synchronization can fail for many reasons.

```mermaid
flowchart TD
    FAILURE[Sync Failure]

    FAILURE --> NETWORK[Network Failure]
    FAILURE --> AUTH[Authentication Failure]
    FAILURE --> VALIDATION[Invalid Payload]
    FAILURE --> CRYPTO[Decryption Failure]
    FAILURE --> DB[Local Transaction Failure]
    FAILURE --> VERSION[Version Conflict]
    FAILURE --> SERVER[Remote Failure]

    NETWORK --> RETRY[Retryable]
    SERVER --> RETRY

    AUTH --> ACTION[Requires Authentication Recovery]
    VALIDATION --> INVESTIGATE[Requires Investigation]
    CRYPTO --> RECOVERY[Recovery / Historical Data Handling]
    VERSION --> RECONCILE[Conflict Reconciliation]
```

A robust sync engine therefore needs to distinguish between:

**retryable failures** and **structural failures**.

Blind retries cannot solve corrupted or incompatible data.

---

# 20. Fresh Device Synchronization

A newly installed device has no meaningful local synchronization state.

Its first synchronization is therefore closer to initialization than incremental synchronization.

```mermaid
flowchart TD
    A[Fresh Device]
        --> B[Authenticate]

    B --> C[Resolve User + Store Access]

    C --> D[Initialize Local Database]

    D --> E[Pull Authorized Remote Dataset]

    E --> F[Validate / Decrypt]

    F --> G[Persist Locally]

    G --> H[Store Synchronization Watermark]

    H --> I[Normal Incremental Sync Begins]
```

Fresh-device behavior must be tested separately from existing-device synchronization because the assumptions are different.

---

# 21. Recovery and Historical Data

Real systems eventually accumulate records created by older versions of the application.

That creates compatibility challenges.

Historical data may contain:

* older schemas;
* legacy encryption formats;
* missing metadata;
* partially migrated values;
* records produced by previous synchronization logic.

The synchronization system therefore requires recovery paths that distinguish between:

```mermaid
flowchart TD
    RECORD[Historical Record]

    RECORD --> VALID[Valid Current Format]
    RECORD --> LEGACY[Known Legacy Format]
    RECORD --> BROKEN[Malformed / Unknown]

    VALID --> NORMAL[Normal Processing]

    LEGACY --> MIGRATE[Controlled Migration / Recovery]

    BROKEN --> QUARANTINE[Do Not Silently Corrupt Data]
```

The safest response to unknown data is not always "try harder to parse it."

Sometimes the correct behavior is to stop, preserve evidence, and recover deliberately.

---

# 22. Testing Strategy

Synchronization requires more than happy-path unit tests.

Important cases include:

### Normal behavior

* create locally then push;
* update remotely then pull;
* no-op when versions match.

### Connectivity

* connection drops before request;
* connection drops after server accepts request;
* device remains offline for extended periods.

### State conflicts

* local newer;
* remote newer;
* both modified from the same base version.

### Isolation

* Store A cannot pull Store B data;
* store identifiers cannot be accidentally removed from payloads.

### Recovery

* malformed encrypted field;
* legacy plaintext;
* partially migrated record;
* invalid synchronization watermark.

### Device lifecycle

* fresh installation;
* restored backup;
* returning device;
* removed/re-authorized device.

---

# 23. Synchronization Invariants

Several invariants help reason about correctness.

### Invariant 1

A successful local user action must not depend on immediate cloud availability unless the operation inherently requires the server.

### Invariant 2

A failed push must not silently mark unsynchronized local changes as synchronized.

### Invariant 3

The pull watermark must never advance beyond safely processed remote state.

### Invariant 4

A retry must not create duplicate logical records.

### Invariant 5

Store isolation must hold locally and remotely.

### Invariant 6

Synchronization must not silently convert encrypted sensitive data into plaintext.

### Invariant 7

Unknown or malformed historical records must not silently overwrite healthy data.

---

# Key Lesson

Offline-first architecture provides a strong user experience because application availability is decoupled from network availability.

But this creates a distributed system.

```mermaid
flowchart LR
    OFFLINE[Offline Capability]
        --> MULTIPLE[Multiple Independent States]

    MULTIPLE --> SYNC[Need for Synchronization]

    SYNC --> CONFLICT[Conflicts]
    SYNC --> RETRY[Retries]
    SYNC --> VERSION[Versioning]
    SYNC --> RECOVERY[Recovery]
    SYNC --> TESTING[More Complex Testing]
```

The architecture is therefore a trade:

> **Less operational dependency on the network in exchange for substantially more synchronization complexity.**

For OptixSuite, that trade is worthwhile because store operations should remain available even when connectivity is unreliable.
