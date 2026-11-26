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
    - [Overview](#overview)
    - [Packaging](#packaging)
    - [MachineSet API](#machineset-api)
    - [Template changes](#template-changes)
    - [Network interface templates](#network-interface-templates)
    - [Example](#example)
    - [Member identity and naming](#member-identity-and-naming)
    - [Machine health](#machine-health)
    - [Provisioning](#provisioning)
    - [Recreating a Machine](#recreating-a-machine)
    - [Rollout](#rollout)
    - [Volume expansion](#volume-expansion)
    - [Encryption](#encryption)
    - [Scale-down and deletion](#scale-down-and-deletion)
    - [Status](#status)
- [Security Considerations](#security-considerations)
- [Testing Strategy](#testing-strategy)
- [Alternatives](#alternatives)

## Summary

Introduce `MachineSet`, an optional operator that keeps a requested number of
VM workloads running from one template, following the Kubernetes StatefulSet
pattern.

Each workload in the set, called a member, has a stable identity. It runs on
one Machine at a time, which remains replaceable. The member's data disks and
network identity survive when its Machine is recreated, and are kept when the
set is scaled down or deleted. The root disk is recreated from the image with
each new Machine, so durable data belongs on data disks.

When a member's Machine is deleted or evicted, MachineSet recreates it. When the
template changes in a way that requires a new Machine, such as a new compute
class or image, MachineSet replaces the members' Machines one at a time. Disks
can be expanded, and their encryption keys rotated, without replacing
Machines.

## Motivation

IronCore's `Machine` is a replaceable execution resource.
[IEP-21](https://github.com/ironcore-dev/enhancements/blob/main/ieps/21-machine-eviction.md)
defines eviction through Machine deletion and leaves recreation to an external
owner. Today, direct API consumers have to recreate evicted Machines and
reattach their disks and network interfaces themselves.

Several Machine properties, such as the MachineClass and the root image, are
immutable. Changing them means creating a new Machine. A workload that should
outlive a single Machine needs its data disks, its addresses, and its declared
configuration to carry over to the new Machine.

[Issue #65](https://github.com/ironcore-dev/enhancements/issues/65) discusses
this ownership boundary, and
[issue #75](https://github.com/ironcore-dev/enhancements/issues/75) outlines the
MachineSet proposed here. The proposal covers a singleton VM as well as a
homogeneous group, such as three broker VMs with the same configuration but
independent data disks and addresses. A heterogeneous application uses
several MachineSets.

### Goals

- Maintain a requested number of members from one template, each running on
  one Machine at a time, and recreate a member's Machine when it has been
  deleted or evicted.
- Give each member a stable identity and its own data Volumes and
  NetworkInterfaces, which survive the recreation of its Machine.
- Apply template changes that require a new Machine by rolling the set,
  replacing one member's Machine at a time.
- Allow root and data Volumes to be expanded, and their encryption keys
  rotated, in place.
- Keep members' data Volumes and network identity when the set is scaled
  down or deleted.

### Non-Goals

These are out of scope for this version. Deferred features are described under
[Alternatives](#alternatives).

- Updating Machines in place, live migration, or CPU and memory hotplug. A
  Machine's MachineClass is immutable, and IronCore has no API for live
  migration or hotplug. A recreated Machine is a new object with a new UID.
- Health checks beyond a new Machine coming up, and recovering unhealthy VMs.
  This depends on more detailed Machine health reporting; today a Machine
  reports only its state (see [Machine health](#machine-health)).
- Per-member configuration, heterogeneous sets, or adding and removing data
  Volumes and NetworkInterfaces after the set has been created.
- Reserving compute capacity.
- Snapshots, backup, and restore.
- Taking over Machines that are managed by other controllers.

## Proposal

### Overview

Introduce a namespaced `MachineSet` resource, reconciled by one controller. The
controller works only through IronCore's public APIs; placement, provisioning,
attachment, and power remain with the existing controllers, poollets, and
providers.

A member is one VM workload of the set. It is identified by its ordinal and
has its own data Volumes and NetworkInterfaces, which last as long as the
member. Its Machine and root Volume are its current execution: they are
replaced on rollout and recreated after deletion or eviction.

The root Volume is created from the image with each new Machine and does not
keep its content. Workloads must keep data that has to survive on data Volumes,
so that a new OS image can be applied by rolling the set.

For a set `brokers` with a data Volume template `data` and a NetworkInterface
template `primary`, the member with ordinal `2` consists of:

| Object | Name | Created by | Deleted by |
| --- | --- | --- | --- |
| Machine | `brokers-2` | MachineSet | MachineSet on rollout or scale-down; a user; eviction; garbage collection on set deletion |
| Root Volume | `brokers-2-root` | Machine ephemeral-volume controller | Garbage collection, with its Machine |
| Data Volume | `brokers-data-2` | MachineSet | A user |
| NetworkInterface | `brokers-primary-2` | MachineSet | A user |
| Ephemeral Prefix, VirtualIP | (per NetworkInterface) | NetworkInterface controllers | With their NetworkInterface |

The controller labels every Machine, data Volume, and NetworkInterface it
creates with:

| Label | Value |
| --- | --- |
| `machineset.<group>/name` | Name of the MachineSet. |
| `machineset.<group>/uid` | UID of the MachineSet. |
| `machineset.<group>/ordinal` | Ordinal, such as `2`. |
| `machineset.<group>/template` | Template name (Volumes and NetworkInterfaces only). |
| `machineset.<group>/revision` | Template revision (Machines only). |

Machines have a controller owner reference to the MachineSet. Data Volumes and
NetworkInterfaces have none, so garbage collection never deletes them.

### Packaging

MachineSet is packaged as an optional operator.
Deployments that need it install it into the cluster that serves the IronCore API.
It consists of:

- a `MachineSet` CustomResourceDefinition with `status` and `scale`
  subresources;
- the controller;
- a validating webhook for the rules in this document;
- RBAC that lets the controller manage Machines, Volumes, and NetworkInterfaces,
  and read MachineClasses, MachinePools, VolumeClasses, and Secrets.

The CRD needs its own API group. IronCore's aggregated API server serves
`v1alpha1.compute.ironcore.dev` through an APIService, so a CRD cannot share
that group and version. This document writes the group as `<group>`.

### MachineSet API

The template is a dedicated MachineSet type, not a full `MachineSpec`. The
controller translates it into each Machine's fields. Field names follow the
Machine and Volume APIs where a direct counterpart exists.

| MachineSet field | Machine field set by the controller |
| --- | --- |
| `spec.replicas` | Number of Machines; default `1`. |
| `spec.template.metadata` | `metadata.labels`, `metadata.annotations` |
| `spec.template.spec.machineClassRef` | `spec.machineClassRef` |
| `spec.template.spec.machinePoolRef` | `spec.machinePoolRef` (optional) |
| `spec.template.spec.machinePoolSelector` | `spec.machinePoolSelector` (optional) |
| `spec.template.spec.tolerations` | `spec.tolerations` (optional) |
| `spec.template.spec.ignitionRef` | `spec.ignitionRef` (optional) |
| `spec.template.spec.efiVars` | `spec.efiVars` (optional) |
| `spec.template.spec.ephemeralVolume.name` | `spec.volumes[0].name`; default `root` |
| `spec.template.spec.ephemeralVolume.image` | `spec.volumes[0].ephemeral.volumeTemplate.spec.dataSource.osImage.image` |
| `spec.template.spec.ephemeralVolume.size` | `spec.volumes[0].ephemeral.volumeTemplate.spec.resources.storage` |
| `spec.template.spec.ephemeralVolume.volumeClassRef` | `spec.volumes[0].ephemeral.volumeTemplate.spec.volumeClassRef` |
| `spec.template.spec.ephemeralVolume.encryption.secretRef` | `spec.volumes[0].ephemeral.volumeTemplate.spec.encryption.secretRef` |
| `spec.volumeTemplates[i]` | `spec.volumes[i+1].volumeRef` to Volume `<set>-<name>-<ordinal>`, whose spec is `spec.volumeTemplates[i].spec` |
| `spec.networkInterfaceTemplates[i]` | `spec.networkInterfaces[i].networkInterfaceRef` to NetworkInterface `<set>-<name>-<ordinal>` |
| (none) | `spec.power: On` on creation |

A data Volume template has a `name`, optional `metadata`, and a `spec` that is
passed through to each Volume. All Volume spec fields are supported, except
`claimRef` and `unclaimable`, which IronCore sets. A NetworkInterface template has a
`name`, optional `metadata`, and a `spec` described in
[Network interface templates](#network-interface-templates).

Template names are unique within their list, and attachment names are unique
on the Machine.

A template field is allowed if every member can share its value, or if each
member gets its own value from it, such as an address allocated per member.
Values that only one member can use are limited to sets of at most one
member; see [Network interface templates](#network-interface-templates).
Machine fields not listed above can be added under the same rule.
`guestConfig.hostname` is not supported: IronCore passes it to the guest as
is, so a template would give every member the same hostname.

### Template changes

| Behaviour | Fields |
| --- | --- |
| Rolls the set (see [Rollout](#rollout)) | `machineClassRef`, `machinePoolRef`, `machinePoolSelector`, `tolerations`, `ignitionRef`, `efiVars`, `ephemeralVolume.image`, `ephemeralVolume.volumeClassRef` |
| Applied to existing Volumes | `ephemeralVolume.size` and `volumeTemplates[].spec.resources.storage` (increase only, see [Volume expansion](#volume-expansion)); `ephemeralVolume.encryption` and `volumeTemplates[].spec.encryption` (see [Encryption](#encryption)) |
| Applies at once | `replicas` |
| Immutable | everything else, including template metadata, `ephemeralVolume.name`, all other `volumeTemplates[].spec` fields, the NetworkInterface templates, and adding, removing, or reordering templates |

Volume changes never roll the set. Other Volume template changes are not pushed
to existing Volumes; only a size increase and changes needed for ongoing
operation, such as encryption-key rotation, are. Whether a Volume accepts them
depends on its VolumeClass and the Volume API; the IronCore API server
validates them, not MachineSet. New Volumes are always created from the current
template.

Only references are compared. A change to the content behind a reference, such
as new ignition data in the same Secret or a new image pushed to the same tag,
does not create a revision. Each new Machine uses the content that is current
when it is created.

`machinePoolRef` is optional. If it is set, MachineSet does not check whether
that pool can reach the members' data Volumes and Networks.

### Network interface templates

All members are created from one template, so a field must either yield a
distinct value per member or be safe to share. Fixed values work only when
there is a single member.

| NetworkInterface field | Allowed |
| --- | --- |
| `networkRef` | Any set. Shared by all members. |
| `ipFamilies` | Any set. |
| `ips[].ephemeral.prefixTemplate` with `prefixLength` and `parentRef` or `parentSelector` | Any set. Each member allocates its own address. |
| `prefixes[].ephemeral.prefixTemplate` with `prefixLength` and `parentRef` or `parentSelector` | Any set. Each member allocates its own prefix. |
| `virtualIP.ephemeral.virtualIPTemplate` with `reclaimPolicy: Delete` or unset | Any set. Each member gets its own VirtualIP. |
| `attributes` | Any set. Copied to each member. |
| `metadata.labels`, `metadata.annotations` | Any set. Copied to each member. |
| `ips[].value`, `prefixes[].value`, `prefixTemplate.spec.prefix` | Only while `replicas` is at most `1`. |
| `virtualIP.virtualIPRef` | Only while `replicas` is at most `1`. |
| `virtualIP.ephemeral.virtualIPTemplate` with `reclaimPolicy: Retain` | Rejected. |
| `machineRef`, `providerID` | Rejected. Set by the system. |

A fixed IP or prefix names one specific address, so it cannot be fanned out
to several members. IronCore's IPAM allocates a fixed `prefixTemplate.spec.prefix`
only while it is free, so a second member's allocation would never succeed. A
fixed `value` is passed to the network as is, so every member would get the
same address. A VirtualIP reference points to one VirtualIP, which can be
assigned to only one NetworkInterface. A set that uses one of these fields
cannot be scaled beyond one member.

A member's network identity consists of its NetworkInterfaces, their IPs,
prefixes, and VirtualIP, and the order of the interfaces. Reattaching the same
NetworkInterfaces to every new Machine keeps this identity. The guest MAC
address is not part of the IronCore API and is not preserved.

### Example

```yaml
apiVersion: <group>/v1alpha1
kind: MachineSet
metadata:
  name: brokers
spec:
  replicas: 3
  template:
    spec:
      machineClassRef:
        name: broker-medium
      ignitionRef:
        name: broker-ignition
      ephemeralVolume:
        name: root
        image: registry.example.org/images/broker:v1
        size: 20Gi
        volumeClassRef:
          name: fast
  volumeTemplates:
    - name: data
      spec:
        volumeClassRef:
          name: durable-expandable
        resources:
          storage: 100Gi
  networkInterfaceTemplates:
    - name: primary
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
```

Each of the three members gets its own data Volume, private address, and
public VirtualIP. These survive the recreation of the member's Machine and the
deletion of the set. Each new Machine boots from a new root Volume.

### Member identity and naming

A member is identified by its ordinal, from `0` to `replicas - 1`. Its objects
have deterministic names: its current Machine is `<set>-<ordinal>`, the root Volume is
`<set>-<ordinal>-<root name>`, and its data Volumes and NetworkInterfaces are
`<set>-<template>-<ordinal>`. The set name and template names must be short
enough for all derived names to be valid.

Before creating anything for an ordinal, MachineSet checks every name that the
ordinal needs: its Machine, data Volumes, and NetworkInterfaces. A name is
available if no object of that name exists, or if the object carries this set's
UID label. If any name is taken by another object, MachineSet creates nothing
for that ordinal and reports a `NameConflict` error naming the object. It never
adopts, overwrites, or replaces such an object.

| Case | Situation | Result |
| --- | --- | --- |
| Reuse after scale-down | Set `brokers` (UID `A`) is scaled from 3 to 2. Machine `brokers-2` is deleted; `brokers-data-2` and `brokers-primary-2` remain with label `uid=A`. The set is scaled back to 3. | All names for ordinal 2 are available: the Machine does not exist, and the Volume and NetworkInterface carry `uid=A`. MachineSet creates `brokers-2` and attaches them. |
| Recreated set | Set `brokers` (UID `A`) is deleted, and a new set `brokers` (UID `B`) is created. `brokers-data-0` remains with label `uid=A`. | Ordinal 0 reports `NameConflict` for `brokers-data-0`, and nothing is created. Because ordinals are provisioned in order, no other ordinal is provisioned either. After the user deletes the remaining resources, provisioning starts. |
| Foreign object | A user has created a Volume `brokers-data-1` without MachineSet labels. | Ordinal 1 reports `NameConflict`; the Volume is left untouched. |

Because a Machine name can exist only once, a member can never have two
Machines at the same time.

### Machine health

In this version, a new Machine has come up once MachineSet has observed it with
`status.state: Running` and every entry of `status.volumes` and
`status.networkInterfaces` in state `Attached`. MachineSet records this with the
annotation `machineset.<group>/came-up` on the Machine, so the record survives
controller restarts and disappears with the Machine.

A Machine that has not come up, for example one that stays `Pending`, blocks
provisioning, recreation, and rollout of further ordinals. A Machine that has
come up no longer blocks them, even if it is later shut down or reports
another state.

MachineSet uses this only to order its work and to report status. It does not
delete, restart, or power on a Machine based on its state. This will be
extended as Machine health reporting becomes more detailed.

### Provisioning

Members are provisioned in ascending ordinal order. For each member, MachineSet:

1. Checks that all names are available (see
   [Member identity and naming](#member-identity-and-naming)) and that the
   MachineClass, VolumeClasses, and Secrets the template references exist.
   Otherwise it reports the error and stops.
2. Creates the data Volumes and NetworkInterfaces, or reuses those that carry
   this set's UID label.
3. Creates the Machine with the ephemeral root Volume, the data Volumes, and the
   NetworkInterfaces, powered on.

The next member is provisioned once the previous Machine has come up (see
[Machine health](#machine-health)).

### Recreating a Machine

When a requested member has no Machine, for example because a user deleted it
or IEP-21 evicted it, MachineSet:

1. Waits until the old Machine and its root Volume are gone, and no data Volume
   or NetworkInterface of the member is still claimed by the old Machine.
2. Provisions the member as described above, from the current template.

If several Machines are missing, they are recreated one at a time in ascending
ordinal order, each after the previous one has come up.

A Machine on an unreachable pool cannot finish deleting, so the member waits.
Removing a Machine's finalizers manually bypasses this safeguard.

If the new Machine cannot be placed, the member reports `WaitingForCapacity`
and keeps its data Volumes and NetworkInterfaces.

### Rollout

The rollout fields (see [Template changes](#template-changes)) are hashed into a
revision. Each Machine carries the revision it was created from in the
`machineset.<group>/revision` label. A Machine whose revision differs from the
template's is outdated.

Missing Machines are recreated first. Then MachineSet rolls the outdated
Machines one at a time, highest ordinal first. For each, it:

1. Waits until every other Machine of the set has come up (see
   [Machine health](#machine-health)).
2. Checks that the names are available and that the MachineClass, MachinePool,
   Secrets, and VolumeClass that the current template references exist. If
   any is missing, it reports `MissingReference` and leaves the Machine
   running.
3. Deletes the Machine and waits until it and its root Volume are gone and its
   data Volumes and NetworkInterfaces are released.
4. Creates the Machine under the same name from the current template, with the
   same data Volumes and NetworkInterfaces, powered on.
5. Waits until the new Machine has come up, then continues with the next
   outdated Machine.

A rollout takes one Machine out of service at a time and recreates its root
disk. Outdated Machines that are powered off are rolled too; the new Machine is
created powered on. There is no capacity check before the deletion; if the new
Machine cannot be placed, the rollout waits with `WaitingForCapacity`.

If the template changes during a rollout, the rollout continues towards the
latest revision. A new Machine that never comes up stops the rollout at its
ordinal with `WaitingToComeUp`.

### Volume expansion

Increasing `ephemeralVolume.size` or `volumeTemplates[].spec.resources.storage`
expands the existing root or data Volumes of every requested member in place.
MachineSet sets `spec.resources.storage` on each Volume whose request is
smaller. Data Volumes of ordinals outside the desired range are expanded when
the member is scaled up again. New Volumes are created with the new size.

MachineSet updates both the root Volume and the ephemeral Volume template in
the Machine's spec, so the Machine shows the values in effect. The Machine API
accepts this change, and it does not change the Machine's revision. The
template change alone does not resize the Volume, because IronCore's
ephemeral-volume controller only creates the Volume and never updates it.

MachineSet rejects a decrease. It does not check the VolumeClass: if the
VolumeClass has `resizePolicy: Static`, the IronCore API server rejects the
Volume update. MachineSet then reports `VolumesUpdated=False` with reason
`ExpansionRejected`, naming the Volume, and retries when the template changes.
Expanding the guest file system is up to the workload.

### Encryption

`ephemeralVolume.encryption.secretRef` and a data Volume template's
`spec.encryption.secretRef` are applied when a Volume is created. Changing
either rotates the key of the existing Volumes in place, without a rollout:
MachineSet sets the new Secret on each root or data Volume of the set,
including data Volumes of ordinals outside the desired range, and the Volume
and storage layer rotates the key. For root Volumes, MachineSet also updates the
ephemeral Volume template in the Machine's spec, as for size. MachineSet does
not delete the old Secret.

Rotation depends on the
[Volume encryption-key rotation design](https://github.com/ironcore-dev/enhancements/pull/62).
Today the Volume API treats `spec.encryption` as immutable, so the API server
rejects the change, and MachineSet reports `VolumesUpdated=False` with reason
`EncryptionChangeRejected`. Once #62 is implemented, the same template change
rotates the keys without changes to MachineSet.

All members share the Secrets named in the template. MachineSet checks that
they exist but never reads them.

### Scale-down and deletion

Scale-down deletes Machines one at a time, highest ordinal first, and waits
until each Machine is gone before deleting the next. Their data
Volumes and NetworkInterfaces remain, unclaimed and labelled. Scaling up again
reuses them, with their addresses and VirtualIPs.

Deleting the MachineSet deletes its Machines through garbage collection and
nothing else. Data Volumes, NetworkInterfaces, and their Prefixes and VirtualIPs
remain, count against quotas, and keep public addresses allocated. In this
version, users clean them up manually:

```shell
kubectl delete volumes,networkinterfaces -l machineset.<group>/uid=<set-uid>
```

A new MachineSet with the same name gets a new UID. It cannot use the remaining
resources and reports `NameConflict` until they are deleted.

### Status

| Field | Meaning |
| --- | --- |
| `observedGeneration` | Generation observed by the controller. |
| `replicas` | Existing Machines of the set. |
| `readyReplicas` | Machines in state `Running` with all volumes and network interfaces `Attached`. |
| `updatedReplicas` | Machines whose revision label matches `updateRevision`. |
| `updateRevision` | Revision of the current template. |
| `conditions` | `Ready`, `Progressing`, and `VolumesUpdated`. The reason and message name the blocking ordinal or object. |

Condition reasons include `WaitingToComeUp`, `WaitingForCapacity`,
`WaitingForRelease`, `NameConflict`, `MissingReference`, `ExpansionRejected`,
and `EncryptionChangeRejected`.

Members, including those outside the desired range, can be listed by the
`machineset.<group>/uid` label across Machines, Volumes, and NetworkInterfaces.

## Security Considerations

- **Acting for the user.** Permission to create a MachineSet effectively grants
  creating Machines, Volumes, NetworkInterfaces, and public VirtualIPs in that
  namespace. Existing quotas apply to everything the controller creates.
- **Retained resources.** Volumes and NetworkInterfaces outlive the set. Their
  data and public addresses stay allocated until a user deletes them.
- **Resource reuse.** MachineSet only reuses Volumes and NetworkInterfaces that
  carry its UID label. A user who can create those resources and set labels in
  the namespace could still place one; the namespace is the trust boundary, as
  for StatefulSet PersistentVolumeClaims.
- **One Machine per member.** The stable Machine name prevents two Machines for
  the same member, and MachineSet waits until the old Machine is gone before
  creating a new one. An unreachable pool never leads to a second Machine.
- **Exclusive inputs.** Fixed IPs, fixed prefixes, and VirtualIP references are
  limited to singletons, so a template cannot hand the same address to several
  members.
- **Secrets.** Members share the template's encryption and ignition Secrets.
  MachineSet never reads their content.

## Testing Strategy

- **Unit and envtest:** template validation and immutability, the
  NetworkInterface field rules, the translation into Machine fields, revision
  hashing, name checks, expansion, and status.
- **End to end** (IronCore e2e suite), on a multi-host environment with
  network-attached Volumes and IronCore networking: provisioning, a rollout
  for each rollout field, recreation after eviction, scale-down
  and scale-up, set deletion and recreation, Volume expansion with
  `ExpandOnly` and `Static` VolumeClasses, and capacity shortage. After each
  recreation, check that data Volumes, IPs, prefixes, VirtualIPs, and interface
  order are unchanged and the root Volume is new. Check that a rollout replaces
  one Machine at a time, highest ordinal first; that Volume size and key
  changes never roll the set; that a Machine that never comes up stops the
  rollout; and that a Machine shut down after coming up does not.
  Check encryption-key rotation once #62 is implemented.
- **Fault injection:** controller restarts at each step, concurrent template
  edits, name conflicts, and an unreachable pool. Each test asserts that no
  member has two Machines and that no data Volume or NetworkInterface is
  deleted or duplicated.

## Alternatives

Alternatives under "Deferred" are excluded from this version but may be added
later. Alternatives under "Not chosen" were considered and set aside for this
design.

### Deferred

#### Other rollout strategies

StatefulSet also offers `partition` for canaries, `maxUnavailable` for
parallel updates, and `OnDelete`, which leaves the timing to the user. This
version rolls one Machine at a time.

#### Retention policies

MachineSet could offer policies to delete Volumes and NetworkInterfaces on
scale-down or set deletion, as StatefulSet does with
`persistentVolumeClaimRetentionPolicy`. This version always retains them and
leaves cleanup to the user. A policy can be added with `Retain` as the default.

#### Persistent root disk

The root disk could be a retained Volume instead of an ephemeral one. A new OS
image could then no longer be applied by rolling the set; the OS on each member
would have to be updated from inside the guest. Keeping the root disk ephemeral
and durable data on data Volumes lets an image update roll the set like any
other change. If consumers cannot keep their data on data Volumes, this
decision will be revisited.

#### Replacement on pool health timeout

MachineSet could replace Machines on a pool that stays unreachable. Unreachable
does not mean stopped, so this could run two copies of a member against the
same disks. It requires a proper fencing mechanism, which guarantees that the
old Machine has stopped and can no longer write to its disks before a new
Machine starts. IronCore does not have one yet; once it does, MachineSet can
use it to replace Machines on unreachable pools.

### Not chosen

#### Per-member API object

An earlier draft proposed a controller-owned `MachineSetMember` that recorded
each member's resources, UIDs, and replacement progress. Stable names and
labels make that state derivable from existing objects, so the extra API, its
RBAC, and its garbage-collection rules are not needed.

#### External orchestration only

Consumers can already recreate Machines and reattach resources themselves, but
each would repeat the same IronCore-specific lifecycle and failure handling.
