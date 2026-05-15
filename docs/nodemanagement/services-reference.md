---
eyebrow: 'Docs · NodeManagement'
lede:    'The four NodeManagement services — AddNodes, DeleteNodes, AddReferences, DeleteReferences. Per-service inputs, outputs, and the failure paths that exist on open62541.'

see_also:
  - { href: './overview.md',                    meta: '4 min' }
  - { href: '../servers/open62541-nm.md',       meta: '4 min' }

prev: { label: 'Overview',    href: './overview.md' }
next: { label: 'Policies and modes', href: '../security-and-auth/policies-and-modes.md' }
---

# Services reference

The four NodeManagement services. All four target the
`open62541-nm` server (port 24840).

## `AddNodes`

Create new nodes in the address space.

### Request — `AddNodesItem` per node

| Field                     | Type                | Description                                     |
| ------------------------- | ------------------- | ----------------------------------------------- |
| `parentNodeId`            | ExpandedNodeId      | Existing node the new one is referenced from    |
| `referenceTypeId`         | NodeId              | The reference between parent and new node       |
| `requestedNewNodeId`      | ExpandedNodeId      | Desired NodeId (server may override)            |
| `browseName`              | QualifiedName       | New node's BrowseName                            |
| `nodeClass`               | NodeClass enum       | `Object`, `Variable`, `Method`, …               |
| `nodeAttributes`          | ExtensionObject     | Class-specific attributes (see below)           |
| `typeDefinition`          | ExpandedNodeId      | TypeDefinition for `Object` / `Variable` nodes  |

### `nodeAttributes` — class-specific

The `nodeAttributes` ExtensionObject is one of:

| NodeClass    | Attributes type                                          |
| ------------ | -------------------------------------------------------- |
| `Object`     | `ObjectAttributes` (DisplayName, EventNotifier, …)        |
| `Variable`   | `VariableAttributes` (DataType, ValueRank, Value, AccessLevel, …) |
| `Method`     | `MethodAttributes` (Executable, …)                        |
| `ObjectType` | `ObjectTypeAttributes`                                    |
| `VariableType` | `VariableTypeAttributes`                                |
| `ReferenceType` | `ReferenceTypeAttributes`                              |
| `DataType`   | `DataTypeAttributes`                                      |
| `View`       | `ViewAttributes`                                          |

Most client libraries provide builders that hide the
ExtensionObject encoding. For low-level wire tests, you'd
construct it manually.

### Response — `AddNodesResult` per node

| Field              | Type           | Description                            |
| ------------------ | -------------- | -------------------------------------- |
| `statusCode`       | StatusCode      | `Good` or `Bad_*`                      |
| `addedNodeId`      | NodeId          | The NodeId the server actually assigned |

### Common failures

| Status                         | Cause                                                |
| ------------------------------ | ---------------------------------------------------- |
| `Bad_NodeIdExists`             | The requested NodeId is already in the address space |
| `Bad_ParentNodeIdInvalid`      | Parent doesn't exist                                 |
| `Bad_ReferenceTypeIdInvalid`   | Reference type isn't a valid NodeId                   |
| `Bad_NodeClassInvalid`         | NodeClass + attributes don't match                    |
| `Bad_TypeDefinitionInvalid`    | TypeDefinition required and missing or wrong          |
| `Bad_BrowseNameInvalid`        | Empty / malformed BrowseName                          |
| `Bad_UserAccessDenied`         | Caller can't add nodes (not on `open62541-nm` — full access there) |

## `DeleteNodes`

Remove nodes from the address space.

### Request — `DeleteNodesItem` per node

| Field                      | Type     | Description                                              |
| -------------------------- | -------- | -------------------------------------------------------- |
| `nodeId`                   | NodeId   | The node to delete                                       |
| `deleteTargetReferences`   | Boolean  | If `true`, also remove references pointing **to** this node |

### Response — per-node StatusCode

| Status                       | Cause                                              |
| ---------------------------- | -------------------------------------------------- |
| `Good`                       | Node deleted                                       |
| `Bad_NodeIdUnknown`          | Node didn't exist                                  |
| `Bad_UserAccessDenied`       | (not on `open62541-nm`)                            |

### `deleteTargetReferences`

If `true`, the server walks all references in the address space
and removes any that point to the deleted node. Expensive but
keeps the address space coherent.

If `false`, "dangling" references can survive — they target a
NodeId that no longer resolves. Browse calls following those
references return `Bad_NodeIdUnknown` for the target.

Use `true` for normal cleanup; use `false` only when you're
about to delete the source side of those references anyway (a
batch teardown).

## `AddReferences`

Add references between existing nodes.

### Request — `AddReferencesItem` per reference

| Field                | Type            | Description                                       |
| -------------------- | --------------- | ------------------------------------------------- |
| `sourceNodeId`       | NodeId          | Source of the reference                            |
| `referenceTypeId`    | NodeId          | The reference type                                 |
| `isForward`          | Boolean         | Forward or inverse                                 |
| `targetServerUri`    | String          | For cross-server refs (empty for local)            |
| `targetNodeId`       | ExpandedNodeId  | Target                                            |
| `targetNodeClass`    | NodeClass        | What the target is (`Object`, `Variable`, …)      |

### Response — per-reference StatusCode

| Status                       | Cause                                              |
| ---------------------------- | -------------------------------------------------- |
| `Good`                       | Reference added                                    |
| `Bad_SourceNodeIdInvalid`    | Source doesn't exist                               |
| `Bad_TargetNodeIdInvalid`    | Target doesn't exist                                |
| `Bad_ReferenceTypeIdInvalid` | Reference type isn't a valid reference type        |
| `Bad_DuplicateReferenceNotAllowed` | The exact reference already exists           |

### When to add references

The typical case: after `AddNodes` created a new node, you want
it to be **discoverable** via the standard hierarchies. For
example:

```text
1. AddNodes creates ns=1;s=my-method (a Method node).
2. AddReferences adds: ns=1;s=my-folder --HasComponent--> ns=1;s=my-method.
3. AddReferences adds: ns=1;s=my-method --HasTypeDefinition--> ns=0;i=58 (BaseObjectType).
```

Without those references, the method is created but unreachable
via Browse.

In practice, `AddNodes` with the right `parentNodeId` +
`referenceTypeId` handles the parent → new node reference
automatically. You'd use `AddReferences` for **additional**
references beyond that one.

## `DeleteReferences`

Remove references.

### Request — `DeleteReferencesItem` per reference

| Field                       | Type            | Description                                     |
| --------------------------- | --------------- | ----------------------------------------------- |
| `sourceNodeId`              | NodeId          | Source                                          |
| `referenceTypeId`           | NodeId          | Reference type                                  |
| `isForward`                 | Boolean         | Direction                                       |
| `targetNodeId`              | ExpandedNodeId  | Target                                          |
| `deleteBidirectional`       | Boolean         | Also remove the inverse reference                |

### Response — per-reference StatusCode

| Status                          | Cause                                          |
| ------------------------------- | ---------------------------------------------- |
| `Good`                          | Reference removed                              |
| `Bad_NoMatch`                   | The reference didn't exist                     |
| `Bad_SourceNodeIdInvalid`       | Source doesn't exist                            |
| `Bad_ReferenceTypeIdInvalid`    | Reference type bogus                            |

### `deleteBidirectional`

OPC UA references are stored on both nodes (source has a
forward reference; target has an inverse). With
`deleteBidirectional = true`, removing one direction also
removes the other.

For most cleanup, `true` is what you want.

## Batch operations

All four services support **multiple operations per call**:

```text
AddNodes([item1, item2, item3, …])
```

Returns an array of results in the same order. A single bad
item doesn't fail the whole batch — the response shows per-item
status.

This is useful for setting up a complex test fixture in one
round-trip.

## What's not supported on `open62541-nm`

- **Persistent nodes.** Everything you add is lost on container
  restart. Tests should not assume nodes survive between runs.
- **Type model enforcement.** open62541's NodeManagement is
  permissive — it doesn't fully enforce that, e.g., a Variable
  child of an Object matches the Object's TypeDefinition. For
  strict type-model tests, use a server with full type-model
  support (most production servers).

## Where to read next

- [open62541-nm](../servers/open62541-nm.md) — the server.
- [Overview](./overview.md) — context and the canonical test
  pattern.
