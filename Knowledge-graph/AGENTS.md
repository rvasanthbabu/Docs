# AGENTS.md

## Purpose

This repository implements a Jira-to-Amazon Neptune graph synchronization solution.

The system must support:

- Jira record INSERT
- Jira record UPDATE
- Jira record REFRESH / reconciliation
- Jira record DELETE / soft-delete
- Graph relationship creation and maintenance
- Idempotent processing
- Audit and synchronization metadata
- Safe replay of Jira events

Jira is the source of truth. Neptune is the graph projection used for relationship traversal, dependency analysis, and downstream governance use cases.

---

## Graph Model

Neptune is modeled as a property graph.

### Vertex types

- `JiraProject`
- `JiraIssue`
- `JiraUser`
- `JiraComponent`
- `Application`
- `Service`

Additional vertex types may be introduced when required by the target use case.

### Relationship types

- `HAS_ISSUE`
- `ASSIGNED_TO`
- `CREATED_BY`
- `HAS_COMPONENT`
- `BLOCKS`
- `DEPENDS_ON`
- `RELATES_TO`
- `IMPACTS`

Do not create relational-style tables or schemas for Jira entities. Represent entities as vertices, relationships as edges, and attributes as properties.

---

## Identity and ID Strategy

Every source entity must have a deterministic, stable graph ID.

Examples:

```text
jira:project:PROJ
jira:issue:100234
jira:user:john.smith
jira:component:database
```

Never generate random IDs for Jira entities.

Use the immutable Jira/source identifier wherever possible. Jira issue keys such as `PROJ-1234` are useful business identifiers but should not be the only technical identity.

Recommended common properties:

```text
id
sourceSystem
sourceId / jiraId
sourceUpdatedAt
lastSyncedAt
syncVersion
syncHash
isActive
deletedAt
```

---

## Insert / Upsert Rules

Use `MERGE` for deterministic entity creation.

Example:

```cypher
MERGE (i:JiraIssue {
    id: 'jira:issue:100234'
})
SET
    i.jiraId = '100234',
    i.key = 'PROJ-1234',
    i.summary = 'Database production change',
    i.status = 'In Progress',
    i.sourceSystem = 'JIRA',
    i.sourceUpdatedAt = '2026-09-28T20:30:00Z',
    i.lastSyncedAt = '2026-09-28T20:35:00Z',
    i.isActive = true;
```

The same Jira event may arrive more than once. Processing the event repeatedly must not create duplicate vertices.

---

## Relationship Rules

Create relationships using `MERGE` whenever the relationship should be unique.

Example:

```cypher
MATCH (p:JiraProject {
    id: 'jira:project:PROJ'
})
MATCH (i:JiraIssue {
    id: 'jira:issue:100234'
})
MERGE (p)-[:HAS_ISSUE]->(i);
```

For Jira-to-Jira relationships:

```cypher
MATCH (i1:JiraIssue {
    id: 'jira:issue:100234'
})
MATCH (i2:JiraIssue {
    id: 'jira:issue:100567'
})
MERGE (i1)-[:BLOCKS]->(i2);
```

Do not create duplicate edges when the same event is replayed.

---

## Update Rules

Updates must match the existing vertex by stable ID.

Example:

```cypher
MATCH (i:JiraIssue {
    id: 'jira:issue:100234'
})
SET
    i.status = 'Done',
    i.sourceUpdatedAt = '2026-09-28T21:00:00Z',
    i.lastSyncedAt = '2026-09-28T21:05:00Z',
    i.syncVersion = 16,
    i.isActive = true;
```

### Important

The synchronization service should:

1. Validate the Jira event.
2. Identify the entity using the stable source ID.
3. Compare source version/timestamp when available.
4. Ignore stale events.
5. Update only fields supplied by the authoritative payload.
6. Update synchronization metadata after successful processing.
7. Maintain idempotency.

Do not allow an older Jira event to overwrite newer Neptune data.

---

## Delete Rules

Prefer soft deletion for synchronization.

Example:

```cypher
MATCH (i:JiraIssue {
    id: 'jira:issue:100567'
})
SET
    i.isActive = false,
    i.deletedAt = '2026-09-28T21:10:00Z',
    i.lastSyncedAt = '2026-09-28T21:10:00Z';
```

Soft deletion preserves graph history and supports reconciliation/audit requirements.

Use:

```cypher
DETACH DELETE
```

only when permanent deletion is explicitly required.

---

## Relationship Deletion

When Jira removes a relationship, delete the edge rather than deleting either entity.

Example:

```cypher
MATCH (i1:JiraIssue {
    id: 'jira:issue:100234'
})-[r:BLOCKS]->(i2:JiraIssue {
    id: 'jira:issue:100567'
})
DELETE r;
```

The synchronization service must distinguish between:

- Entity deletion
- Relationship deletion
- Entity update
- Relationship addition

---

## Refresh / Reconciliation

Jira webhooks should be treated primarily as change notifications.

A periodic refresh must reconcile Jira with Neptune.

Recommended flow:

```text
Jira
 |
 +-- Webhook ------------------+
 |                              |
 +-- Scheduled Full Refresh ---+
                                |
                                v
                         Sync / Diff Engine
                                |
                  +-------------+-------------+
                  |                           |
               New/Changed                 Missing
                  |                           |
                  v                           v
             Neptune Upsert             Soft Delete
```

Refresh processing must be idempotent.

The reconciliation process should identify:

- New Jira records missing from Neptune
- Jira records with changed properties
- Missing/deleted Jira records
- Missing relationships
- New relationships
- Removed relationships
- Stale or failed synchronization records

---

## Event Processing

Recommended processing sequence:

```text
Jira Event
   |
   v
Validate
   |
   v
Extract entity type + source ID
   |
   v
Normalize payload
   |
   v
Check version / timestamp
   |
   v
MERGE vertex
   |
   v
SET properties
   |
   v
MERGE / DELETE relationships
   |
   v
Update sync metadata
   |
   v
Audit success
```

Events should be safe to retry.

Failed events should be routed to a dead-letter mechanism such as an SQS DLQ.

---

## Suggested AWS Architecture

```text
                         JIRA
                           |
              +------------+------------+
              |                         |
          Webhooks                  Scheduled
              |                    Reconciliation
              v                         v
       API Gateway/Lambda         EventBridge
              |                         |
              +------------+------------+
                           |
                           v
                          SQS
                           |
                           v
                    Sync Processor
                   Lambda / ECS
                           |
              +------------+------------+
              |                         |
          Jira API                   S3 Raw
          validation                snapshots
              |
              v
        Transformation
              |
              v
          Amazon Neptune
              |
              +------------------+
              |                  |
           Audit             Monitoring
           / DLQ             CloudWatch
```

---

## Data Quality Rules

Before writing to Neptune:

- Validate required source IDs.
- Validate entity type.
- Normalize timestamps to UTC.
- Normalize Jira status and issue type values according to the agreed mapping.
- Reject malformed events.
- Do not create a vertex with a missing deterministic ID.
- Do not create relationships when either endpoint cannot be resolved.
- Record failed records for retry/reconciliation.

---

## Synchronization Metadata

Each synchronized vertex should preferably contain:

```text
sourceSystem    = "JIRA"
sourceId        = Jira immutable ID
sourceUpdatedAt = timestamp from Jira
lastSyncedAt    = timestamp when Neptune was successfully updated
syncVersion     = source/event version where available
syncHash        = normalized payload hash where useful
isActive        = true/false
deletedAt       = deletion timestamp when applicable
```

These fields are used for troubleshooting, reconciliation, and preventing stale updates.

---

## Query Examples

### Find a Jira issue

```cypher
MATCH (i:JiraIssue {
    id: 'jira:issue:100234'
})
RETURN i;
```

### Find issues assigned to a user

```cypher
MATCH (i:JiraIssue)-[:ASSIGNED_TO]->(u:JiraUser)
WHERE u.id = 'jira:user:john.smith'
RETURN i;
```

### Find blocked issues

```cypher
MATCH (a:JiraIssue)-[:BLOCKS]->(b:JiraIssue)
RETURN a.key, b.key;
```

### Find all active Jira entities

```cypher
MATCH (n)
WHERE n.sourceSystem = 'JIRA'
  AND coalesce(n.isActive, true) = true
RETURN labels(n), n.id, n.lastSyncedAt;
```

---

## Coding Guidelines

- Prefer small, composable synchronization functions.
- Keep Jira API access separate from Neptune persistence logic.
- Keep graph transformation logic separate from transport logic.
- Use configuration rather than hard-coded Jira project names or IDs.
- Never embed credentials in source code.
- Use AWS Secrets Manager or an equivalent secure secret store.
- Use structured logging.
- Include Jira source ID and correlation/event ID in logs.
- Make retry behavior explicit.
- Avoid non-idempotent graph writes.
- Add unit tests for transformation and identity logic.
- Add integration tests for Neptune writes.
- Test duplicate, stale, out-of-order, and replayed events.

---

## Security

Never commit:

- Jira API tokens
- AWS access keys
- Neptune credentials
- Private keys
- Production URLs containing secrets
- Personally sensitive data not required by the graph model

Use IAM roles and managed AWS authentication where supported.

Apply least-privilege permissions.

---

## Testing Requirements

At minimum test:

1. New Jira issue creates exactly one vertex.
2. Reprocessing the same issue does not create a duplicate.
3. Updating an issue changes the existing vertex.
4. An older event does not overwrite a newer version.
5. A new relationship is created once.
6. A removed relationship is deleted.
7. A deleted Jira issue is soft-deleted.
8. Full refresh discovers missing records.
9. Full refresh does not duplicate existing records.
10. Failed events can be retried safely.
11. Invalid source IDs are rejected.
12. Neptune failures do not acknowledge the source event prematurely.

---

## Definition of Done

A Jira synchronization change is complete when:

- Entity identity is deterministic.
- Insert/upsert is idempotent.
- Update behavior is tested.
- Delete behavior is defined.
- Relationship handling is tested.
- Refresh/reconciliation behavior is covered.
- Retry behavior is safe.
- Audit metadata is written.
- Logging and monitoring are implemented.
- Secrets are externalized.
- Tests pass.
- No duplicate graph entities are produced during event replay.

---

## Reference Graph

The intended Jira graph should resemble:

```text
                         +----------------+
                         | JiraProject    |
                         | PROJ           |
                         +-------+--------+
                                 |
                             HAS_ISSUE
                                 |
                                 v
                         +----------------+
                         | JiraIssue      |
                         | PROJ-1234     |
                         +--+----------+-+
                            |          |
                     ASSIGNED_TO      BLOCKS
                            |          |
                            v          v
                    +------------+  +----------------+
                    | JiraUser   |  | JiraIssue      |
                    | John Smith |  | PROJ-5678      |
                    +------------+  +----------------+
                            ^
                            |
                       CREATED_BY

JiraIssue
    |
    +---- HAS_COMPONENT ----> JiraComponent
    |
    +---- IMPACTS ----------> Application
                                  |
                                  +---- DEPENDS_ON ----> Service
```

This graph model is the baseline for implementing Jira synchronization into Neptune.
