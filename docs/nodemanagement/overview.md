---
eyebrow: 'Docs · NodeManagement'
lede:    'What the NodeManagement service set is, why it''s the gap UA-.NETStandard does not fill, and the test pattern this suite enables.'

see_also:
  - { href: './services-reference.md',         meta: '5 min' }
  - { href: '../servers/open62541-nm.md',       meta: '4 min' }

prev: { label: 'open62541-all-security',   href: '../servers/open62541-all-security.md' }
next: { label: 'Services reference',       href: './services-reference.md' }
---

# NodeManagement overview

The OPC UA spec defines a **NodeManagement service set** —
runtime modification of the address space. Four services:

| Service           | What it does                                              |
| ----------------- | --------------------------------------------------------- |
| `AddNodes`        | Create new nodes (Objects, Variables, Methods, …)         |
| `DeleteNodes`     | Remove nodes (optionally also their incoming references)   |
| `AddReferences`   | Add references between existing nodes                      |
| `DeleteReferences` | Remove references                                          |

These are the four services your client library calls to
**dynamically shape an OPC UA server's address space at runtime**.

## Why this suite exists

UA-.NETStandard — the reference stack `uanetstandard-test-suite`
is built on — **does not implement these services over the wire**.
You can call `AddNodes` on a client connected to any
UA-.NETStandard server and get back `Bad_ServiceUnsupported`
(or similar).

That's a problem for client libraries that:

- **Expose the NodeManagement API** in their public surface.
- **Want integration tests** that drive that API against a real
  server.
- **Want regression coverage** for the wire-encoding of
  `AddNodes` payloads (which are non-trivial — `ExtensionObject`-
  laden, type-definition-aware).

open62541 implements NodeManagement fully. `open62541-nm`
(port 24840) exposes it without security or auth getting in the
way.

## When NodeManagement actually matters

It's a less-common scenario than `Read`/`Write`/`Subscribe`. The
typical use cases:

| Use case                                            | Why NodeManagement                                   |
| --------------------------------------------------- | ---------------------------------------------------- |
| Edge gateway dynamically modelling discovered devices | New devices → new node trees, on the fly           |
| Configuration-driven server                          | Operator changes config → server reshapes itself    |
| Test fixture creation                                | Test creates nodes, runs assertions, deletes them   |
| Wire-spec validation                                 | Verify the `AddNodes` payload encoder round-trips    |

The first two are real production scenarios. Most production
clients don't need NodeManagement — they consume an address
space someone else built.

## The test pattern

The canonical test against `open62541-nm`:

```text
1. Connect to opc.tcp://localhost:24840 (anonymous, no security).
2. AddNodes — create the fixture(s) the test needs.
3. Exercise feature under test (browse, read, write, call, subscribe).
4. Assert.
5. DeleteNodes — clean up (or leave for the next test).
6. Disconnect.
```

The address space is **empty** at connect time. Tests bring
their own data.

## What you can create

| NodeClass         | Use case                                              |
| ----------------- | ----------------------------------------------------- |
| `Object`          | A folder or grouping in the address space             |
| `Variable`        | A typed value (with `DataType` + `ValueRank`)         |
| `Method`          | A callable method                                     |
| `ObjectType`      | Define a new type for instantiation                    |
| `VariableType`    | Define a new variable type                            |
| `ReferenceType`   | Define a new reference type                            |
| `DataType`        | Define a custom data type                              |
| `View`            | Define a filtered browse view                          |

`Object`, `Variable`, and `Method` are the most common in
practice.

## What you can reference

A new node typically uses one of these reference types between
it and its parent:

| ReferenceType         | Used between                                  |
| --------------------- | --------------------------------------------- |
| `Organizes`           | Folder → folder, folder → leaf                 |
| `HasComponent`        | Object → child variable / child object         |
| `HasProperty`         | Variable → property (e.g. EURange)             |
| `HasTypeDefinition`   | Instance → its TypeDefinition                  |

For the standard reference type NodeIds (`ns=0;i=35` for
`Organizes`, etc.) see the OPC UA spec or your client library's
constants.

## A complete round-trip

```text
1. Connect.

2. AddNodes (one node):
   - parentNodeId:    ns=0;i=85   (Objects folder)
   - referenceTypeId: ns=0;i=35   (Organizes)
   - requestedNewNodeId: ns=1;s=my-folder
   - browseName:      1:MyFolder
   - nodeClass:       Object

3. Browse(ns=0;i=85) → expect "MyFolder" in results.

4. AddNodes (one variable, child of MyFolder):
   - parentNodeId:    ns=1;s=my-folder
   - referenceTypeId: ns=0;i=47   (HasComponent)
   - requestedNewNodeId: ns=1;s=my-folder.my-value
   - browseName:      1:MyValue
   - nodeClass:       Variable
   - dataType:        Int32
   - value:           42

5. Read(ns=1;s=my-folder.my-value) → 42.

6. Write(ns=1;s=my-folder.my-value, 99) → Good.

7. Read(ns=1;s=my-folder.my-value) → 99.

8. DeleteNodes([ns=1;s=my-folder.my-value, ns=1;s=my-folder]).

9. Browse(ns=0;i=85) → "MyFolder" no longer present.

10. Disconnect.
```

Every step is a separate test assertion.

## Behaviour notes

- **Empty namespace.** `ns=1` is the writable test namespace —
  use it for all your test nodes. The server doesn't enforce
  this (`ns=2..N` also work) but `ns=1` is the convention.
- **`requestedNewNodeId` may be ignored.** The server can choose
  its own NodeId and return it in the response. The example test
  above assumes the request is honoured (which `open62541-nm`
  does in practice).
- **Cleanup is your responsibility.** The server doesn't
  auto-purge nodes between sessions. A test that creates nodes
  and doesn't delete them leaves them for the next test.

## Where to read next

- [Services reference](./services-reference.md) — the four
  services' wire signatures and inputs.
- [open62541-nm](../servers/open62541-nm.md) — the server
  itself.
