---
title: MachineSet

iep-number: NNNN

creation-date: 2026-09-17

status: implementable

authors:

- "@shawnsarwar"

reviewers: []

---

# IEP-NNNN: MachineSet

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [API resources and controllers](#api-resources-and-controllers)
    - [MachineSet API](#machineset-api)
    - [MachineSetMember API](#machinesetmember-api)
    - [Example](#example)
    - [Resource identity and ownership](#resource-identity-and-ownership)
    - [Network identity](#network-identity)
    - [Provisioning and power state](#provisioning-and-power-state)
    - [Restoring a member's execution](#restoring-a-members-execution)
    - [Compute rollout](#compute-rollout)
    - [Scale-down and deletion](#scale-down-and-deletion)
    - [Volumes and encryption](#volumes-and-encryption)
    - [Status](#status)
    - [Relation to host fencing](#relation-to-host-fencing)
- [Security Considerations](#security-considerations)
- [Testing Strategy](#testing-strategy)
- [Alternatives](#alternatives)

## Summary

Introduce `MachineSet` to maintain a requested number of durable VM members from
one shared template. Each member retains its identity, disks, and network
identity while its underlying `Machine` can be replaced.

Users configure `MachineSet`, which coordinates membership and updates.
A controller-created `MachineSetMember` records each member's resources and
progress while its Machine is absent, including after scale-down retains them.
Deleting the set deletes its members and owned resources and releases external
ones.

We propose an optional upstream IronCore controller component with associated
API resources. Packaging and integration decisions are left to maintainers.

## Motivation

IronCore's `Machine` is a replaceable execution resource.
[IEP-21](https://github.com/ironcore-dev/enhancements/blob/main/ieps/21-machine-eviction.md)
defines eviction through Machine deletion and leaves recreation to an external
owner. Direct API consumers currently need to manage this lifecycle themselves.

A long-running VM needs continuity beyond the lifetime of an individual Machine.
When that Machine is replaced, the workload should retain its root and data
disks, its network identity, and its declared configuration.

Users may also need to change a VM's compute size. In IronCore, this means
selecting another MachineClass. Because a Machine's class reference is
immutable, this requires a replacement Machine that reuses the workload's disks
and network identity.

[Issue #65](https://github.com/ironcore-dev/enhancements/issues/65) discusses this
ownership boundary and a MachineSet abstraction. This proposal covers both a
singleton VM with a persistent root disk and a homogeneous group, such as three
broker VMs with the same configuration but independent disks and addresses.
A heterogeneous application would use multiple MachineSets.

### Goals

- Maintain a requested number of VM workloads whose configuration, disks, and
  network identity survive Machine replacement.
- Provision independent disks and network interfaces for each member from one
  shared template.
- Recreate a member's Machine when it has been deleted, including by eviction.
- Allow CPU/RAM sizing changes through sequential Machine replacement.
- Make resource retention and deletion explicit during scale-down and when
  deleting MachineSet.
- Expose each member's current Machine, resources, addresses, and progress
  through the API.

### Non-Goals

- Live migration, CPU/memory hotplug, or preservation of the Machine UID.
- Snapshots, backup/restore, reimaging, disk resize, or automatic rollback.
- Application-aware preparation, quiescence, or health checks.
- Detecting host loss or replacing Machines on an unreachable pool. See
  [Relation to host fencing](#relation-to-host-fencing).
- Reserving or pre-checking compute capacity.
- Per-member configuration or heterogeneous sets.
- Preserving the guest-visible MAC address.
- Managing encryption keys, including rotation and destruction.
- Taking over Machines already managed by other controllers.

## Proposal

### API resources and controllers

Introduce namespaced `MachineSet` and `MachineSetMember` resources in
`compute.ironcore.dev/v1alpha1`. MachineSet works only through IronCore's public
resource APIs; placement, provisioning, power, and attachment remain with the
existing controllers, poollets, and providers.

| Component | Responsibility |
| --- | --- |
| MachineSet controller | Membership, shared configuration, provisioning and rollout order, aggregate status. |
| MachineSetMember controller | Durable resource bindings, current Machine, operation progress, and safe handoff between Machines. |

Both reconcilers run in one controller component. Field names and examples are
proposed shapes, not final structures.

### MachineSet API

| Field | Meaning |
| --- | --- |
| `spec.replicas` | Desired member count; default `1`. Powered-off members still count. |
| `spec.template` | Shared Machine template, including `machineClassRef`, `power`, placement, tolerations, `guestConfig`, and attachments. Attachments may use `volumeTemplateRef` and `networkInterfaceTemplateRef`. |
| `spec.volumeTemplates` | Named templates for standalone per-member Volumes. |
| `spec.networkInterfaceTemplates` | Named templates for per-member NetworkInterfaces. |
| `spec.resourceRetentionPolicy.whenScaledDown` | `Retain` (default) or `Delete`, applied when a member leaves the desired range. |

Only `replicas`, `power`, `machineClassRef`, and the retention policy are
mutable. Everything that defines where a member runs and what it is attached
to is immutable from creation: placement and tolerations, attachment layout,
Network, image and bootstrap references, hostname, and template storage
capacity. Changing `power` updates existing Machines; changing
`machineClassRef` starts a [compute rollout](#compute-rollout).

Inputs that are exclusive by nature are accepted only while `replicas` is one,
including on later scale-up: fixed IPs and prefixes, VirtualIP references, and
`guestConfig.hostname`. A set that uses them cannot scale beyond one. Without
`guestConfig.hostname`, the guest hostname is not managed by MachineSet. A
singleton may also reference externally owned Volumes and NetworkInterfaces.

Image and bootstrap (`ignitionRef`) references are fixed, but their content may
evolve externally. New root Volumes and new Machines use the content available
when they are created; content changes do not trigger a rollout or reimage
existing roots, and the guest is not guaranteed to rerun first-boot logic on a
retained root.

Durable disks are standalone Volumes: created per member from
`volumeTemplates`, or, for a singleton, referenced externally. A `localDisk` may
be used for scratch space; it belongs to one Machine, is created anew for each
Machine, and does not keep its data across replacement. Ephemeral Volume and
NetworkInterface sources are rejected, because IronCore deletes them with their
Machine.

### MachineSetMember API

A member occupies a zero-based position (its ordinal) within its MachineSet. Its
own UID distinguishes a retained member from a new member later created at the
same ordinal. A revision identifies the set's Machine configuration; today only
a change of `machineClassRef` creates a new revision.

| Field | Meaning |
| --- | --- |
| `spec.machineSetRef` | Originating MachineSet name and UID. |
| `spec.ordinal` | Logical position. |
| `spec.lifecycle` | `Active`, `Retained`, or `Reclaiming`. |
| `spec.targetRevision` | Configuration selected for this member. |
| `status.appliedRevision` | Configuration whose Machine reached readiness. |
| `status.machineRef` | Current Machine name and UID, when present. |
| `status.volumes`, `status.networkInterfaces` | UID-bound resource references and observed state. |
| `status.operation` | Current operation, its phase, and the predecessor Machine. |
| `status.conditions` | Readiness, progress, and blocking reasons. |

Because the member records its progress independently of the Machine, a
controller restart in the middle of a replacement is designed to resume without
losing or duplicating resources. Consumers can read members; only the controller
can create or modify them. Deleting a `Retained` member is the explicit way to
reclaim it and its owned resources.

### Example

```yaml
apiVersion: compute.ironcore.dev/v1alpha1
kind: MachineSet
metadata:
  name: brokers
spec:
  replicas: 3
  template:
    spec:
      machineClassRef:
        name: broker-medium
      power: "On"
      volumes:
        - name: root
          volumeTemplateRef:
            name: root
      networkInterfaces:
        - name: primary
          networkInterfaceTemplateRef:
            name: primary
  volumeTemplates:
    - metadata:
        name: root
      spec:
        volumeClassRef:
          name: durable-block
        resources:
          storage: 40Gi
        dataSource:
          osImage:
            image: registry.example.org/images/broker:v1
  networkInterfaceTemplates:
    - metadata:
        name: primary
      spec:
        networkRef:
          name: application-network
        ipFamilies:
          - IPv4
        ips:
          - ephemeral:
              prefixTemplate:
                spec:
                  ipFamily: IPv4
                  prefixLength: 32
                  parentRef:
                    name: application-prefix
        virtualIP:
          ephemeral:
            virtualIPTemplate:
              spec:
                type: Public
                ipFamily: IPv4
                reclaimPolicy: Delete
  resourceRetentionPolicy:
    whenScaledDown: Retain
```

Each of the three members gets its own root Volume, NetworkInterface, private
address, and public VirtualIP, all of which survive replacement of its Machine.

### Resource identity and ownership

The member records the name and UID of every resource it uses, and whether it
created the resource or it was supplied externally. Creation is idempotent per
member, so retries never allocate a second set of resources.

Resources are bound by UID, not by name. Resources of another owner are never
adopted, and recreating a resource or a MachineSet under the same name does not
transfer a binding. A missing or conflicting resource is reported as an error,
never silently replaced with a new, empty one.

Created resources are owned by the member, never by the replaceable Machine.
They survive Machine replacement and retained scale-down and are cleaned up with
the set. External and shared resources are released, never deleted.

### Network identity

A member's network identity is:

- its private IPs and prefixes;
- its VirtualIP associations;
- its Network, and its interface names and their order in the Machine spec;
- for a singleton, its hostname.

MachineSet keeps this identity by retaining the member's NetworkInterface and
attaching it to each successor. Its ephemeral IPs, prefixes, and non-`Retain`
VirtualIPs are owned by it and come along. Interface names, their order, and
the hostname come from the template.

VirtualIPs follow their source form:

| Source | Accepted | Lifecycle |
| --- | --- | --- |
| Ephemeral, `reclaimPolicy: Delete` (or unset) | Any set | Owned by the member's NetworkInterface; kept across replacement, deleted when the member is reclaimed. |
| Ephemeral, `reclaimPolicy: Retain` | Never | Not owned by the NetworkInterface; no MachineSet lifecycle is defined. |
| Reference to an existing VirtualIP | Singleton only | Attached and released, never deleted. |

Some properties belong to a particular Machine and are not preserved. The guest
MAC address is not part of the IronCore API, and how interfaces appear inside the
guest depends on the provider. Images should not bind network configuration to
a MAC address.

A successor is created only after the predecessor Machine is fully deleted and
the NetworkInterface claim is released. Any remaining host-side cleanup is left
to the networking provider.

### Provisioning and power state

Members are provisioned in ascending ordinal order, one at a time. The next
member starts when the previous one is ready at its requested power state, so a
member that cannot be placed holds back later members; set status names the
member it is waiting on. Reactivating a retained member reuses its resources.

`power: Off` keeps a member requested but powered off; it is not scale-down and
does not release resources. A power change updates the existing Machine and
never replaces it. A shutdown from inside the guest does not change the
requested power state and does not cause MachineSet to replace the Machine.

### Restoring a member's execution

When a requested member's Machine is deleted by someone else, including by
IEP-21 eviction, MachineSet waits for the deletion to complete and then creates
a successor Machine with the member's retained disks and network interfaces.
Successors and newly provisioned members always use the current target
revision. MachineSet observes the deletion itself; the evicting controller does
not need to track it. A Machine on an unreachable pool cannot finish deleting,
so the member keeps waiting. Manually removing that Machine's finalizer bypasses
this safeguard.

Nothing else causes MachineSet to replace a Machine. In particular, a pool that
becomes unreachable (`Ready=Unknown`) is reported on the member but does not
trigger replacement, because it does not show that the predecessor has stopped.
Power-off, guest shutdown, and scale-down are not failures.

Successors are placed by the existing scheduler within the template's placement
constraints. These constraints must also express where the member's Volumes and
network are reachable; MachineSet does not discover this itself. While the
successor has no pool assigned, for example for lack of capacity, the member
reports `WaitingForCapacity`, keeps all its resources, and continues once the
successor is placed. MachineSet does not migrate or substitute resources.

### Compute rollout

Changing `machineClassRef` replaces members' Machines one at a time, highest
ordinal first. If any member is already unavailable, the rollout pauses until it
is available again. For each member:

1. Record the target configuration, and check that the new class and all
   referenced inputs exist.
2. Delete the predecessor Machine and wait until its deletion is complete and
   its attachments are released.
3. Create the successor with the new class and the retained resources.
4. Wait for readiness, record the applied configuration, and move on.

Replacement keeps the requested power state and involves downtime for that
member; a rollout takes at most one member out of service at a time. There is no
capacity check before the predecessor is deleted: if capacity is short
afterwards, the member waits as described above. A newer class change applies
to the member in progress, and a successor that has not been placed yet is
recreated with the latest class, so a bad class change can be corrected.
MachineSet tracks the MachineClass by UID; if the class is deleted or
recreated, dependent work stops with a visible reason and running Machines are
left alone.

### Scale-down and deletion

Scale-down removes the highest ordinals first.

| Action | Result | Member lifecycle |
| --- | --- | --- |
| Scale down with `Retain` | Remove the Machine; keep the member and its resources. | `Retained` |
| Scale up a retained ordinal | Reactivate the member with its resources and the current configuration. | `Active` |
| Scale down with `Delete` | Remove the Machine, then the member's owned resources and the member. | `Reclaiming` |
| Delete a `Retained` member | Delete its owned resources, then the member. | `Reclaiming` |
| Delete the MachineSet | Remove all Machines, members, and owned resources, including retained ones. | `Reclaiming` |

The retention policy applies when a member leaves the desired range. Changing
the policy later does not affect members that are already retained; they are
reclaimed by deleting them or the set. Scaling to zero with `Retain` releases
all compute while keeping the durable resources.

Once set deletion begins, no new Machines are created. Deletion always removes
Machines and releases attachments before deleting resources, and a finalizer
keeps the set until cleanup is complete.

### Volumes and encryption

Template storage capacity is immutable and applies to new Volumes. Replacement
keeps each member's existing Volumes as they are. Volume expansion is left to a
separate proposal.

MachineSet attaches encrypted Volumes like any other and leaves key management
to the Volume and storage layer, including the
[Volume encryption-key rotation design](https://github.com/ironcore-dev/enhancements/pull/62).
MachineSet stores no keys or key references in member state and never changes a
Volume's encryption configuration. An encryption Secret named in a Volume
template is passed to every member's Volume, so members share it; MachineSet
never reads it.

### Status

MachineSet status reports `readyReplicas`, `updatedReplicas`, `pendingReplicas`,
the target configuration, the last progress time, `Ready` and `Progressing`
conditions, and a compact list of members with their ordinal, UID, and current
operation phase. Details stay on each member, and members can be listed directly
by the set's UID.

Member status reports the current Machine, each resource with its UID and
observed state (including addresses and Volume capacity), and conditions. The
reason `PoolUnreachable` means the current Machine's pool is `Ready=Unknown`;
`WaitingForCapacity` means a new Machine has no pool assigned yet.

```yaml
status:
  appliedRevision: revision-1
  machineRef:
    name: brokers-2-execution-1
    uid: "<machine-uid>"
  volumes:
    - name: root
      ownership: Managed
      volumeRef:
        name: brokers-2-root
        uid: "<volume-uid>"
  networkInterfaces:
    - name: primary
      ownership: Managed
      networkInterfaceRef:
        name: brokers-2-primary
        uid: "<interface-uid>"
      ips:
        - "10.0.0.23"
      virtualIP: "203.0.113.23"
```

### Relation to host fencing

Recovering a member after host loss needs a fencing mechanism, which IronCore
does not have today and this proposal does not define. If IronCore gains a
signal that a Machine is safely stopped, MachineSet may use it as an additional
trigger for creating a successor; that interface belongs to that future work.

## Security Considerations

- **Access.** Users manage MachineSets in their namespace. MachineSetMembers are
  written only by the controller; users can read them and delete a `Retained`
  member to reclaim it.
- **Acting for the user.** All references are namespace-local. Permission to
  create MachineSets effectively grants creation of Machines, Volumes,
  NetworkInterfaces, and public VirtualIPs in that namespace, so grant it
  accordingly. Existing quotas apply to everything the controller creates.
- **Ownership.** Another owner's resources are never adopted, and external or
  shared resources are never deleted.
- **One Machine per member.** MachineSet never creates a second Machine for a
  member while the previous one exists, and an unreachable pool alone never
  authorises a successor. Machines reference Volumes and NetworkInterfaces by
  name; after creating a successor, MachineSet checks that the resources it
  claimed carry the bound UIDs and deletes the successor on a mismatch.
  Teardown on the host is left to the existing poollets and providers. Removing
  a Machine's finalizer by hand voids this guarantee.
- **Exclusive inputs.** Fixed IPs, VirtualIP references, and hostnames are
  limited to singletons, so a template cannot hand the same address or name to
  several members.
- **Secrets.** Member state contains no keys, credentials, or Secret content.

## Testing Strategy

- **Unit and envtest:** template validation and immutability, UID binding and
  conflict handling, idempotent creation across controller restarts, retention
  policy, and status.
- **End to end** (IronCore e2e suite): on a multi-host environment with
  network-attached Volumes and IronCore networking: provisioning, power changes,
  scale-down and reactivation, set deletion, replacement after deletion and
  eviction, capacity shortage, and class rollout. After each replacement, check
  that disks, IPs, prefixes, VirtualIPs, and interface order are unchanged. An
  unreachable or offline pool never triggers replacement.
- **Fault injection:** controller interruption at each step, concurrent edits,
  and conflicting resources. Each test asserts that a member never has two
  Machines and that no resource is duplicated or deleted early.

## Alternatives

### External orchestration only

Consumers can already implement retention and sequential replacement
themselves, but each would repeat the same IronCore-specific lifecycle and
failure handling. MachineSet offers a common implementation without changing
existing consumers.

### StatefulSet-style bookkeeping without MachineSetMember

A StatefulSet reconstructs its slots from the set, Pods, PVCs, and revision
history. Our members must also record network resources, owned versus external
references, UIDs, and progress while no Machine exists, including after scale
to zero. A typed member record keeps that discoverable at the cost of an extra
API. It is a representation trade-off, not a claim that the StatefulSet pattern
cannot be extended.

### Durable Machine identity

Keeping one Machine and changing its placement or size would change IronCore's
immutable-placement and eviction model. A set/member lifecycle keeps Machine
replaceable and leaves providers unchanged.

### Replacement on pool health timeout

MachineSet could replace Machines on a pool that stays unreachable for some
time. Unreachable does not mean stopped, so this could run two copies of a
member against the same disks. MachineSet therefore does not replace Machines on
unreachable pools; recovery after host loss is out of scope (see
[Relation to host fencing](#relation-to-host-fencing)).

### Capacity check before replacement

MachineSet could require free capacity before deleting a predecessor. A check
is not a reservation and would duplicate scheduler logic, so this proposal
reports `WaitingForCapacity` instead and keeps all resources.
