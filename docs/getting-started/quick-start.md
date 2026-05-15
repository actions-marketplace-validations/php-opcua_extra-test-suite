---
eyebrow: 'Docs · Getting started'
lede:    'One minute from docker compose up to a connected client. Both servers, the simplest happy path.'

see_also:
  - { href: './installation.md',                        meta: '3 min' }
  - { href: '../nodemanagement/overview.md',            meta: '4 min' }
  - { href: '../security-and-auth/policies-and-modes.md', meta: '4 min' }

prev: { label: 'Installation',   href: './installation.md' }
next: { label: 'Servers · Overview', href: '../servers/overview.md' }
---

# Quick start

## Start the suite

<!-- @code-block language="bash" label="terminal" -->
```bash
docker compose up -d
```
<!-- @endcode-block -->

Both servers come up in a few seconds.

## Connect to `open62541-nm` (port 24840)

The NodeManagement server. Anonymous full access, no security.

```text
endpoint:        opc.tcp://localhost:24840
security policy: None
security mode:   None
identity:        Anonymous
```

Any client can connect with one line:

<!-- @code-block language="bash" label="opcua-cli probe" -->
```bash
opcua-cli browse opc.tcp://localhost:24840 ns=0;i=85
```
<!-- @endcode-block -->

You should see the standard `Objects` folder's children. The
address space is whatever open62541's `ci_server` example ships
— mostly empty by design. **Tests populate it themselves** via
`AddNodes`.

A minimal AddNodes round-trip:

<!-- @code-block language="php" label="addnodes round-trip" -->
```php
use PhpOpcua\Client\OpcuaClient;
use PhpOpcua\Client\Types\NodeId;

$client = new OpcuaClient('opc.tcp://localhost:24840');
$client->connect();

// AddNodes
$result = $client->addNodes([
    [
        'parentNodeId'    => 'ns=0;i=85',           // Objects
        'referenceTypeId' => 'ns=0;i=35',           // Organizes
        'requestedNewNodeId' => 'ns=1;s=my-folder',
        'browseName'      => '1:MyFolder',
        'nodeClass'       => 'Object',
    ],
]);

// Browse confirms it exists
$children = $client->browse('ns=0;i=85');
// → contains "MyFolder"

// DeleteNodes
$client->deleteNodes(['ns=1;s=my-folder']);

$client->disconnect();
```
<!-- @endcode-block -->

See [NodeManagement · Services reference](../nodemanagement/services-reference.md)
for the full API.

## Connect to `open62541-all-security` (port 24841)

Every RSA policy, every mode, Anonymous + username/password.

### Anonymous over None

The simplest combination:

```text
endpoint:        opc.tcp://localhost:24841
security policy: None
security mode:   None
identity:        Anonymous
```

### Username over Basic256Sha256 + SignAndEncrypt

```text
endpoint:        opc.tcp://localhost:24841
security policy: Basic256Sha256
security mode:   SignAndEncrypt
identity:        admin / admin123
```

Users:

| Username   | Password      |
| ---------- | ------------- |
| `admin`    | `admin123`    |
| `operator` | `operator123` |
| `viewer`   | `viewer123`   |
| `test`     | `test`        |

Same credentials as `uanetstandard-test-suite`'s userpass server.

The server's certificate is **self-signed** — your client either
needs to pin its fingerprint (TOFU) or use auto-accept mode. See
[Trust posture](../security-and-auth/trust-posture.md).

### Negative paths

| Try                                              | Expected                    |
| ------------------------------------------------ | --------------------------- |
| Wrong password                                    | `Bad_IdentityTokenRejected`  |
| Unknown user                                      | `Bad_IdentityTokenRejected`  |
| Anonymous on a non-None endpoint                  | Server's choice (typically rejected if disabled) |

## Discover endpoints

`GetEndpoints` on the all-security server returns multiple
descriptors — one per (policy, mode) the open62541 build
supports:

<!-- @code-block language="bash" label="endpoint discovery" -->
```bash
opcua-cli get-endpoints opc.tcp://localhost:24841
```
<!-- @endcode-block -->

Expect 6+ endpoints. For the policy matrix, see
[Policies and modes](../security-and-auth/policies-and-modes.md).

## Stop

<!-- @code-block language="bash" label="terminal" -->
```bash
docker compose down
```
<!-- @endcode-block -->

## Where to read next

- [Servers · Overview](../servers/overview.md) — what each
  server does in detail.
- [NodeManagement · Overview](../nodemanagement/overview.md) —
  why this suite exists.
- [Security and authentication](../security-and-auth/policies-and-modes.md) —
  the policy/mode details for the all-security server.
