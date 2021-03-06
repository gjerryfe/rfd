---
authors: Brian Bennett <brian.bennett@joyent.com>, Todd Whiteman
state: predraft
---

# RFD 34 Instance migration

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Introduction](#introduction)
- [Types of Migration](#types-of-migration)
  - [Live Migration](#live-migration)
  - [Offline migration](#offline-migration)
  - [Incremental offline migration](#incremental-offline-migration)
- [The state of migration in SmartOS](#the-state-of-migration-in-smartos)
- [The state of migration in Triton](#the-state-of-migration-in-triton)
- [A vision for the future](#a-vision-for-the-future)
- [Design](#design)
  - [Migration phases](#migration-phases)
    - [Begin phase](#begin-phase)
    - [Sync phase](#sync-phase)
    - [Switch phase](#switch-phase)
  - [User Interfaces](#user-interfaces)
    - [CloudAPI](#cloudapi)
    - [Triton CLI](#triton-cli)
    - [Portal](#portal)
- [Persistent storage](#persistent-storage)
- [Provisioning](#provisioning)
  - [Supporting services](#supporting-services)
  - [Server next reboot date](#server-next-reboot-date)
  - [Affinity constraints](#affinity-constraints)
  - [Traits and reserved CNs](#traits-and-reserved-cns)
  - [Modified instances](#modified-instances)
  - [Network NICs](#network-nics)
  - [Rack Aware Networks](#rack-aware-networks)
  - [Provisioning Questions](#provisioning-questions)
- [Allowing user migrations](#allowing-user-migrations)
- [Re-sync algorithm](#re-sync-algorithm)
- [Limits](#limits)
  - [Open questions for limits](#open-questions-for-limits)
- [Error handling](#error-handling)
  - [Transient errors](#transient-errors)
    - [Resumable zfs send/receive](#resumable-zfs-sendreceive)
  - [Fatal errors](#fatal-errors)
- [Aborting a migration](#aborting-a-migration)
- [Progress](#progress)
  - [Progress events](#progress-events)
- [Scheduling](#scheduling)
- [Migration service](#migration-service)
- [Operator Installation](#operator-installation)
  - [Triton migration agent](#triton-migration-agent)
  - [Triton migration scheduler](#triton-migration-scheduler)
- [Milestones](#milestones)
  - [M1: Initial setup](#m1-initial-setup)
  - [M2: Brand specific migration](#m2-brand-specific-migration)
  - [M3: Update DAPI (CN provisioning/reservation)](#m3-update-dapi-cn-provisioningreservation)
  - [M4: Update CNAPI](#m4-update-cnapi)
  - [M5: Create sdc-migrate tooling](#m5-create-sdc-migrate-tooling)
  - [M6: End user migration](#m6-end-user-migration)
  - [M7: Migration estimate](#m7-migration-estimate)
  - [M8: Migration scheduling](#m8-migration-scheduling)
- [Open Questions and TODOs](#open-questions-and-todos)
  - [Backups](#backups)
  - [Images](#images)
  - [Networking](#networking)
  - [Other](#other)
- [Caveats](#caveats)
- [Tests](#tests)
- [References](#references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Introduction

Migration of instances is a common feature among virtualization platforms. SmartDataCenter is a notable exception. Among public clouds, instance replacement is generally preferred to instance migration.

Nevertheless, at times it has been necessary to migrate instances within the Triton Cloud between compute nodes.

# Types of Migration

There are three types of migration.

- Live migration
- Offline migration
- Incremental offline migration

## Live Migration

This is when an instance compute task is transferred between physical compute nodes. This is generally done using shared storage, requiring only the memory state and execution context to be synchronized leveraging instance "pausing" to briefly quiesce the instance before resuming on the destination compute node. Due to design choices (vis. the exclusion of shared storage) in SDC, and the nature of OS virtualization in SmartOS (containers, not virtual machines*) live migration is not considered for implementation at this time.

\* Even KVM instances are qemu in a *container*.

## Offline migration

Where an instance is stopped completely (i.e., the guest performs a shutdown procedure), the dataset backing the storage for the instance is synced, and the instance is booted on the destination compute node. Offline migration is the most straightforward type of migration and works in almost any circumstance, but has the disadvantage that the instance will be down for the entire time it takes to transfer the storage dataset.

## Incremental offline migration

Occurs when the dataset backing the storage for the instance is synced while the instance is running. Once the dataset is within an acceptable delta on both the source and destination compute node the instance is stopped, a final sync is performed, and the instance is booted on the destination compute node. This allows for both dedicates storage and significantly reduced down time (though, still not as little as live migration).

# The state of migration in SmartOS

ZFS has well known send/receive capabilities. The ability to snapshot a dataset and send incremental datasets makes syncing ZVOLs between compute nodes easy. Similarly, vmadm has undocumented send/receive subcommands that, in addition to performing zfs send, will send the zone metadata. Due to a design decision with the implementation of KVM zones, vmadm send does not support KVM. Migrations can fail when the dataset size is too close to the quota.

# The state of migration in Triton

While vmadm does have the beginning of a migration feature, migrations are not supported in Triton. Out of necessity, some shell scripts have been written that are capable of migrating instances. This is a road fraught with danger, however. On several occasions changes to the Triton stack have rendered the migration script inoperable or worse, dangerous (i.e., incurring data loss).

# A vision for the future

The following is a description of how instance migration might work.

Instance migration should be a first-class supported feature in Triton. This would be comprised of API calls that trigger a workflow to perform the migration unattended. Migrations could be performed on demand, or scheduled. AdminUI would also provide an interface for migrating an instance. Instance migration should use regular DAPI workflows for selecting the destination compute node, or could be specified by an operator to override DAPI selection.

Product messaging via CloudAPI and/or the user portal would notify customers of upcoming scheduled migration, and at their option choose to migrate ahead of the predefined schedule.

# Design

The incremental offline migration will be the only migration type implemented
(it effectively covers the complete offline migration type as well).

Migration will make use of existing provisioning and placement APIs (e.g. CNAPI)
to ensure compatibility with the regular instance creation workflows. This is
so the created instance will be placed on the correct server and that all of the
same services are available (e.g. CNS, Volumes, Networks, etc...) without having
to duplicate the creation of these services.

A migration can be performed immediately or be [scheduled](#scheduling) to run
at a later time.

All migrations will store information in a [migrations](#persistent-storage)
Moray bucket, that includes the details for the migration event (owner,
instance, servers, state, etc...).

When necessary, the migrating instance will be marked with a special "migrating"
vm state to ensure no other operations (e.g. "start", "stop", "reprovision") can
occur on the instance whilst the migration is ongoing (including another
migration attempt).

It will be possible to manually stop/abort a migration, though it may take some
time before the migration process can be safely interrupted and before the
source instance can return to it's prior state.

## Migration phases

All migrations will run in three phases, the user (or operator) can set the
migration to run these phases all at once "automatic", or they can be performed
individually as needed "on-demand". The three phases are "begin", "sync" and
"switch".

Q. Do we want to put a time limit on how long a migration can stay in the "sync"
phase, or do we not care if/can the user will be billed for this?

### Begin phase

This phase starts the migration process, provisions the target instance and
performs the initial full synchronization of data (without stopping the
instance).

1. CNAPI /servers/:server_uuid/vms/:uuid/migrate endpoint will start the
   migration process for this (source) instance by acquiring a ticket/lock on
   the instance, creating the migrations bucket entry, and then changing the
   instance state to "migrating".
2. DAPI will be used to provision a new (target) instance with the same set of
   provisioning parameters (you can think of this as an instance reservation) on
   a different CN (the target CN). Keep the same uuid but flag it as
   'do_not_inventory' to ensure that no systems know/use it yet. This step
   should install the necessary source images on the target CN and/or setup
   necessary supporting zones (e.g. NAT zone). See the
   [provisioning](#provisioning) section for more details on this step.
3. Run a cn-agent task on the source and target CN which will setup a
   communication channel (TCP socket) on the admin network which the two CNs can
   use to perform the migration operation. (this may be combined with step #1
   and step #2)
4. Send initial filesystem data to the target CN. For each filesystem dataset,
   this will create a zfs snapshot and then zfs send that dataset to the waiting
   zfs receive on the target CN.

### Sync phase

This phase synchronizes the zfs datasets of the source instance with the zfs
datasets in the target instance (without stopping the instance).

1. Send the filesystem delta to the target CN. For each filesystem dataset,
   this will create a zfs snapshot and then zfs send the difference from the
   previous snapshot to the zfs receive running on the target CN.
2. When in "automatic" mode, evaluate whether to re-sync (step 5 again) or
   perform the final switch. The migration process will repeat this sync phase
   (with expectedly smaller incremental filesystem updates each time) until
   certain [criteria](#re-sync-algorithm) are met - then it will move on to the
   "switch" phase.

### Switch phase

This phase stops the instance from running, synchronizes the zfs datasets of the
source instance with the zfs datasets in the target instance, moves the NICs
from the source to the target instance, moves control to the target instance and
then restarts the target instance.

1. Stop the source instance (if it was running).
2. Send the final filesystem delta to the target CN. For each filesystem
   dataset, this will create a zfs snapshot and then zfs send the difference
   from the previous snapshot to the zfs receive running on the target CN.
3. Swap NICs - reserve NIC IP addresses in the source instance, delete the
   source NICs and then add same NICs to the target instance.
4. Unregister the source instance from systems (set do_not_inventory on the
   source instance) and ensure it no longer auto-restarts.
5. Register the target instance with systems (remove do_not_inventory from the
   target instance).
6. Start the target instance (if it was previously running).
7. Cleanup (remove source zone, or schedule for later removal).

## User Interfaces

### CloudAPI

The following endpoints will be available for customers:

- ListMigrations (POST /:login/migrations
  Return a list of migrations (completed, running, scheduled, failed, ...).

- MigrateMachine (POST /:login/machines/migrate?action=automatic&affinity=AFF)
  Starts a full "automatic" migration. Affinity rules can be added - see the
  [provisioning](#provisining) section for details.

- CancelMigrateMachine (POST /:login/machines/migrate?action=cancel)
  Cancels the current migration of this instance. See
  [aborting a migration](#aborting-a-migration).

- BeginMigrateMachine (POST /:login/machines/migrate?action=begin&affinity=AFF)
  Starts an on-demand migration (i.e. just the "begin" phase) for an instance.
  Affinity rules can be added - see the [provisioning](#provisining) section for
  details.

- SyncMigrateMachine (POST /:login/machines/migrate?action=sync)
  Runs the "sync" phase for an "on-demand" instance migration.

- SwitchMigrateMachine (POST /:login/machines/migrate?action=switch)
  Runs the final "switch" phase for an "on-demand" instance migration.

- WatchMigrateMachine (POST /:login/machines/migrate?action=watch)
  Will output events for a migration. There can be multiple watchers of a
  migration. See migration [progress events](#progress-events) for details.

- ScheduleMigrateMachine (POST /:login/machines/migrate?action=schedule&affinity=AFF)
  Schedule a migration of this instance. Affinity rules can be added - see the
  [provisioning](#provisining) section for details.

### Triton CLI

    $ triton ls
    SHORTID   NAME        IMG       STATE    FLAGS  AGE
    4a364ec6  myinstance  46b4ceef  running  DF     3d

    $ triton instance migrate --help
    This will allow you to move an instance to a new server. A new
    instance will be reserved and all of the instance filesystems will copied
    across to the new instance. After the filesystems have been initially
    synced, there will be a brief downtime of the instance and re-syncing of
    the filesystems to the new instance, after that the new instance will be
    started on the new server and the old instance will removed.

    $ triton instance migrate --list
    INSTANCE   STATUS   PROGRESS

    $ triton instance migrate 4a364ec6
    This will migrate instance 4a364ec6 (myinstance).
    The migration will take approximately 1h45m.
    Do you wish to continue y/n? y
    Migration of 4a364ec6 has started.
    You can watch the migration using:
      triton instance migrate --watch 4a364ec6
    or you can cancel the migration using:
      triton instance migrate --cancel 4a364ec6

    $ triton ls  # shows 'M' flag meaning the instance is migrating
    SHORTID   NAME        IMG       STATE     FLAGS  AGE
    4a364ec6  myinstance  46b4ceef  running   MDF    3d

    $ triton instance migrate --watch $id
    Migration progress ($migrateUuid):
    ✓ making a reservation (3m18s)
    ✓ synchronizing filesystems (13% ETA 1h33m)
    <Ctrl-C ... migration still continues>

    $ triton instance migrate --watch $id
    Migration progress ($migrateUuid):
    ✓ making a reservation (3m18s)
    ✓ synchronizing filesystems (1h52m07s)
    ✓ stopping instance (19s)
    ✓ re-synchronizing filesystems (1m19s)
    ✓ switching instance information (0m30s)
    ✓ restarting instance (22s)
    Migration was successful.

Scheduling example:

    $ triton instance migrate --schedule "2 days from now" $id
    This will migrate instance $shortId. Do you wish to continue y/n?
    Scheduling migration.
    You can watch the migration using:
      triton instance migrate --watch 4a364ec6
    or you can cancel the migration using:
      triton instance migrate --cancel 4a364ec6
    Initial scheduling complete - migration will commence at $date.

TODO: a way to list migrations.

### Portal

TODO: I'm not sure who looks after this?

# Persistent storage

A record of all migration operations will be stored in a reliable datastore
(e.g. Moray). This will be used to report state to users and operators for past,
ongoing and future migrations.

The following is an example Moray bucket schema to be used for migration
operations:

    migrations {

      uuid: "UUID"
        // The unique migration identifier.

      account_uuid: "UUID"
        // The account who invoked (and thus controls) this migration. The admin
        // user will always be able to control (schedule/abort) all migrations,
        // but non-admin accounts will only be allowed to control their own
        // migrations.

      vm_uuid: "UUID"
        // The uuid of the instance to be migrated.

      source_server_uuid: "UUID"
        // The uuid of the server where the source instance resides.

      target_server_uuid: "UUID"
        // The uuid of the server where the instance will be migrated to.

      state: "string"
        // The state the migration operation is currently in. It can be one of
        // the following states:
        //  "scheduled"  - the migration is scheduled, see "scheduled_timestamp"
        //  "running"    - migration is running, see "job_uuid" and "step"
        //  "pending"    - the "begin" phase (and possibly "sync" phase) has
        //                 ben run - now waiting for a call to "sync"
        //                 or "switch"
        //  "failed"     - migration operation could not complete, see "error"
        //  "aborted"    - user or operator aborted the migration attempt
        //  "successful" - migration was successfully completed, you can use the
        //                 workflow job_uuid to find when the migration started
        //                 and how long it took.

      created_timestamp: "string"
        // The ISO timestamp when the migration was first created. This will
        // also be used for queuing - migrations created first will be closer
        // to the start of the queue than those created later.
        // Example:
        //   2018-08-18T10:55:39.785Z

      scheduled_timestamp: "string"
        // The ISO timestamp when the migration should commence. The migration
        // service will be responsible to start this migration at the
        // appropriate time.
        // Example:
        //   2018-08-19T22:30:00.000Z

      started_timestamp: "string"
        // The ISO timestamp when the migration started. This is useful to see
        // how long a migration has taken.
        // Example:
        //   2018-08-19T22:30:05.142Z

      job_uuid: "UUID"
        // The workflow job uuid (if it was started).

      phase: "string"
        // XXX This could come from the workflow job?
        // The workflow stage that the migration is currently running, one of:
        //  ""                 - nothing running yet
        //  "begin"            - just starting the migration operation
        //  "sync"             - updating filesystems using the "sync" phase
        //  "switch"           - sending final zfs datasets and switching
        //  "finished"         - completed

      num_update_phases: "number"
        // The number of successful "update" phases this migration has
        // performed.

      error: "string"
        // If a migration fails - this is the error message of why it failed.
    }

# Provisioning

There are a lot of factors that need to be considered during the migration
provisioning process.

## Supporting services

Any Triton supporting services that the instance uses (e.g. NAT zones or
Volumes zones) will need to be setup on the target server. If provisioning
using DAPI, this should occur naturally.

## Server next reboot date

Ideally, we want the migrated instance to land on a server that will not be
rebooted soon. If DAPI happens to pick a machine that will be rebooted in
another 2 weeks, that lessons the purpose of the current migration and will
likely aggravate users. We want to make sure the migration provisioning
process takes into account the next reboot attribute, see [RFD 24](../0024),
which if using DAPI should occur naturally.

## Affinity constraints

Affinity hints are only applied at provisioning time. For docker containers,
these hints are persisted as a triton label in the instance metadata. For
other instance types, the hints are not persisted. As such, DAPI will not be
able to honor the original affinity constraints. We should allow the
user/operator to input an optional affinity hint when they initiate/schedule
migrations.

## Traits and reserved CNs

DAPI can handle user specified traits, but there are cases where a trait has
been specified by an operator, and as such, DAPI may not be able to handle that.
In this case the operator will need to either remove that trait from the
instance, or use the sdc-migrate tooling and specify the target server (the
latter will not invoke the DAPI server selection algorithms).

## Modified instances

DAPI uses the package size to determine the allocation. It's possible that
an instance will have gone through an in-place update to increase their
quota, ram, etc and no longer have sizes that match the original package.
We will need to pass additional migration specific instance parameters to
DAPI to specify the wanted size in such a case.

## Network NICs

During the provisioning process, we do not want to allocate any NICs or assign
IP addresses for the target instance. Instead, during the switch phase the NICs
will need to be swapped from the source to the target instance. This NIC swap is
accomplished by first reserving the IP, deleting the source instance NIC and
then adding the NIC to the target instance. We used to have sporadic issues with
duplicate IP addresses after migration when the IP of the instance being
migrated was accidentally handed out for a different provisioning.

## Rack Aware Networks

In the case of a CN being in a rack aware network - the DAPI provisioning
process will need to take into account (if not already done) that the newly
provisioned instance must reside in a CN in the same rack.


## Provisioning Questions

- if migration re-uses the existing cn-agent provisioning process, can some
  parts (e.g. imgadm image install and dataset creation) be skipped?

- what other Triton components will need updating after a migration (i.e. are
  there any Triton services that expect the instance to be on a particular CN)?

# Allowing user migrations

There will be two settings controlling whether a user is allowed to migrate
their instances. These settings can only be set by an operator.

- **CN.traits.user_migration_allowed** (boolean) trait

  - should we also allow the **eol** trait?

  Example:

      $ sdc-cnapi /servers/$CN -X POST -d '{"traits": { "allow_user_migration": true }}'
      $ sdc-migrate allow-cn $CN

- **vm.internal_metadata.user_migration_allowed** (boolean) property on the
  instance can be set to 'true' to allow *user migration* of this instance, a
  missing value (the default) or 'false' will mean the user cannot start a
  migration on this instance. This value overrides the CN trait.

      $ sdc-vmapi "/vms/$VM?action=update" -X POST -d '{ "internal_metadata": { "user_migration_enabled": true }}'
      $ sdc-migrate allow-instance $VM

# Re-sync algorithm

During the incremental filesystem update, the migration system will make a
calculation to see if it needs to continue re-syncing the filesystem or perform
the final sync and switch. The idea is that further re-syncing will reduce the
time needed for the final sync (as the snapshots become smaller and the time
needed to perform the sync becomes faster and faster), and thus reducing the
downtime for the instance.

These are the criteria for when the migration can proceed to the final sync and
switch:

- the incremental snapshot size is less than a configured max size (e.g. 50MB)
- the number of re-sync attempts exceeds a limit (e.g. 10)
- the snapshot size has not reduced significantly in the last N re-sync attempts
  (e.g. last 3 re-syncs)

# Limits

There are limits placed on migrations, to ensure that they do not affect (or
limitedly affect) other tenants on the source/target CN.

The migration limit configuration will be stored in SAPI (initially set during
the migration service installation, or falling back to defaults) as:

    $ sdc-sapi /services?name=migration
    {
      "zfs_send_mbps_limit": 500,  // megabits per second
      "max_migrations_per_cn": 5
    }

The zfs send limits can be enforced by the application (e.g. Node.js), by
adjusting the amount of data being sent/received (piped) to the target.

## Open questions for limits

- can these limits be set (overwritten) on a per CN basis too? Can changes to
  these limits be propagated to migrations that are currently running? Maybe a
  running migration task can check these settings every X minutes and apply
  updates when they change.

- do we need a zfs_receive_bps_limit too - in case of different server
  hardware configurations?

- what happens when we exceed the number of migrations allowed - is that a hard
  error back to the caller? Note that "pending" migrations would not be included
  in this count.

- what happens when it cannot start a scheduled migration at the scheduled time,
  how much leeway is allowed there?

# Error handling

## Transient errors

Transient errors during a migration, like a network failure, or a triton
component failure, will be handled and the migration process will retry to
complete the migration step which failed. The migration workflow will have a
hard coded number of retry operations per migration step with a backoff setting
to allow a flexible delay between the step retry operations.

### Resumable zfs send/receive

The zfs send/receive operations (long/sustained network usage) will make use of
the (resumable zfs send/receive)[#resumable-zfs-sendreceive] feature, so that
the migration operation can resume in the case of an interruption, without
needing to resend all of the zfs data.

In the case of a zfs send/recv failure, a *receive_resume_token* marker is
set in the zfs filesystem, which can be used as an argument to zfs send,
allowing the stream to continue from where it left off:

    $ zfs receive -s  # Keep track of stream position
    $ zfs send -s token  # Continue from this point in the send stream

Note that to use this flag, the storage pool must have the extensible_dataset
feature enabled, which can be queried using:

    $ zpool get -H -o value "feature@extensible_dataset" zones
    enabled

## Fatal errors

Fatal errors, such as:

- source CN is no longer visible/responding

  A "partially" migrated instance will reside on the target CN. This will
  require operator intervention (to bring back the source CN).
  XXX: Can the instance owner delete that failed target instance?

- target CN is no longer visible/responding

  The migration will be aborted and the source instance will be restored to
  it's prior state. A record of the failure is kept in the migrations database.

- when both the source and target CN are not visible/responding

  Not much can be done here - this will require operator intervention.
  If one or both of these CNs come back, then we move to above two items.

# Aborting a migration

There are many factors to consider when aborting a migration, most of which
will depend on the migration phase. There will even be times when it is not
possible to abort at all (in the latter phases of the migration, when the
instance migration is done and the process is now cleaning up).

It will be crucial that the source instance be returned to it's former state,
including the re-instantiation of NICs (in the case they were deleted from the
source instance - see the [network provisioning](#network-nics) notes) and
returning to the source instance to it's previous running/stopped state.

An abort will cause CNAPI to issue stop/abort calls to any ongoing migration
workflow jobs and cn-agent migration tasks, then CNAPI will rollback/cleanup the
migration state and revert the source instance to it's former state (if it was
changed).

Note that it will not be possible to resume an aborted migration, but one could
start a new migration instead.

# Progress

For each migration, we need the ability to show the current state and progress
to the end user.

CNAPI will provide a migration progress endpoint that can be used to monitor
the progress of the migration.

CNAPI will connect with the cn-agent task on the source instance (using a TCP/IP
socket) to retrieve status, filesystem transfer rates and overall progress (e.g.
from zfs send/receive).

## Progress events

The CNAPI progress endpoint will respond with a stream of events that
describe all the completed migration phases and also the current state of the
migration. Whilst connected to the endpoint, a current state event will be
sent back every N seconds (e.g. every 2 seconds).

    event {

      phase: "string"
        // The phase for this event, one of "begin", "sync" or "switch"

      state: "string"
        // The status for the event, e.g. "success", "failed", "running"

      start_timestamp: "string"
        // The ISO timestamp when the phase was started.

      end_timestamp: "string" (optional)
        // The ISO timestamp when the phase finished. When omitted, it indicates
        // that this phase is still running.

      message: "string" (optional)
        // Additional description message for this phase/state.

      current_progress: "number" (optional)
        // Value for how much of the phase has been completed. For the "sync"
        // phase this will be the number of bytes that have been sent to the
        // target.

      total_progress: "number" (optional)
        // Total value for large this phase is. For the "sync" phase this will
        // be the total number of bytes that will need to be sent to the target.
    }

Status event (successful):

    {
      phase: "begin",
      state: "success",
      start_timestamp: "2018-08-18T10:55:39.785Z",
      end_timestamp: "2018-08-18T10:58:01.402Z"
    }

Status event (sync just started running):

    {
      type: "status"
      phase: "sync",
      state: "running",
      start_timestamp: "2018-08-18T10:59:09.338Z",
      current_progress: 0,
      total_progress: 983374382
    }

Status event (sync running):

    {
      type: "status"
      phase: "sync",
      state: "running",
      start_timestamp: "2018-08-18T10:59:10.198Z",
      current_progress: 12039472
      total_progress: 983374382
    }

Status event (sync successful):

    {
      phase: "begin",
      state: "success",
      start_timestamp: "2018-08-18T10:59:10.198Z",
      end_timestamp: "2018-08-18T11:15:44.384Z"
      current_progress: 983374382
      total_progress: 983374382
    }

Status event (switch failure):

    {
      phase: "switch",
      state: "failed",
      message: "Failed to switched NICs to migrated target - aborted"
      start_timestamp: "2018-08-18T11:16:12.600Z"
      end_timestamp: "2018-08-18T11:16:33.974Z"
    }

# Scheduling

A migration can be scheduled to begin running at a configured time/date. In this
case the migration will (immediately on receiving the migration schedule call)
start the migration process which will provision the target instance and perform
the initial zfs snapshot synchronization. At this point the migration process
(if the scheduled time has not yet occurred) will pause, then at the scheduled
time the final shutdown and re-synchronize will occur and will finish the
migration process.

# Migration service

A "triton-migrate-service" zone will be used to control scheduled migrations.
It does this by loading the currently running/scheduled migrations from the
migrations database (Moray bucket) and then periodically checks for updates.

Roles:

- initiates the migration at the scheduled time

Needs to communicates with:

- CNAPI
- VMAPI
- Moray

Note: This service could later be expanded to handle complete CN evacuations,
but that is outside the scope of this RFD.

# Operator Installation

Migration is separated into two main components:

## Triton migration agent

The _triton-migration-agent_ contains the executables (code) needed to perform
an instance migration from one compute node to another. It must be installed in
the GZ on all of the CNs which will support migration (i.e. all CNs). The
triton-migration-agent can be installed and updated via sdcadm:

    $ sdcadm experimental update-agents --latest --all

This agent can be installed anytime, but a migration from or to this CN cannot
occur until this agent has been installed.

## Triton migration scheduler

The scheduler's job is to start migration of instances at a predetermined time.

    $ sdcadm post-setup migration-scheduler

This will create the triton-migration-scheduler zone on a headnode.

This zone is only necessary for migration scheduling, without this zone an
instance migration cannot be scheduled (it will raise an error), but an
on-demand instance migration can still be performed (if the
triton-migration-agent is installed).

# Milestones

## M1: Initial setup

Setup the initial repository and testing infrastructure.

- write brand specific tests for migration
  - create small test images (maybe Alpine based) instead of using Ubuntu?
- initial handling for zfs snapshots and incremental zfs send/receive
- verify zfs send/receive is working for all brands (SmartOS, Docker, LX, Kvm,
  Bhyve)

## M2: Brand specific migration

Concentrate on getting Bhyve (and KVM?) working first. For datasets that are
nearly full, the destination quota may need to be increased slightly, just
enough for all necessary write operations. Ideally, the quota would be reduced
to its original size after migration is complete.

- modify zfs send/receive handling to support the brands needed
- support multiple disks and different zfs filesystem layouts
- add quota workaround for nearly full instances

## M3: Update DAPI (CN provisioning/reservation)

DAPI will be updated to select the target CN and provision (reserve) the
target instance.

- provision (reserve) the target instance
  - choose CN
    - allow allocation to an operator specified server (via sdc-migrate)
    - pass through affinity rules, see the [provisioning](#provisioning) section
    - pass through instance size, see the [provisioning](#provisioning) section
  - allocate/reserve zone
  - install image(s)
  - setup support infrastructure (e.g. Fabric NAT)

## M4: Update CNAPI

CNAPI will trigger and respond to migration calls, using workflow and cn-agent
tasks to control and monitor the migration.

- add CNAPI API migration endpoint
- add workflow tasks for control of the migration states
- create cn-agent tasks to initiate sending/receiving
- other actions on source/target instance cannot occur during a migration
- instance cleanup
- update sdc-clients to add CNAPI migration API wrappers

## M5: Create sdc-migrate tooling

Operators will need a tool to be able to perform a migration from inside of a
CN/Headnode:

    $ sdc-migrate settings [$CN]
    {
        "max_migrations_per_cn": 5
        "zfs_send_mbps_limit": 500
    }
    # Allow updating too?

    $ sdc-migrate list [$CN1 $CN2 $CN3]
    ...<Show ongoing migrations, state, percentage>

    $ sdc-migrate allow-cn $CN
    Success - compute node $CN now allows user migrations.
    $ sdc-migrate disallow-cn $CN
    Success - user migrations are no longer allowed on compute node $CN.

    $ sdc-migrate allow-instance $VM
    Success - instance $VM can now be user migrated.
    $ sdc-migrate disallow-instance $VM
    Success - user migration not allowed for instance $VM.

    $ sdc-migrate migrate $INSTANCE [$CN]
    Migrating $INSTANCE
    ...<progress events>
    ...<progress events>
    <Ctrl-C ... workflow job still continues>

    $ sdc-migrate watch $INSTANCE
    ...<progress events>
    Success - instance $INSTANCE has been migrated to server $CN.

    $ sdc-migrate abort $INSTANCE
    ...<progress events>
    Success - migration was aborted, instance $INSTANCE is now $STATE.

Once a CN (or instance) has been flagged as allowing migrations then the
operator can send an announcement to end users allowing them to migrate their
own instances via CloudAPI or Triton command line. The portal (AdminUI) may make
use of these flags to notify that a CN is going down and that they can migrate
their instances.

## M6: End user migration

Allow users to perform or schedule their own instance migrations.

- add CloudAPI migration APIs
  - this may want to be a workflow job
  - add support for polling migration job
- add Triton cli instance migration support

## M7: Migration estimate

Add APIs that will return an estimate of the time taken to perform a migration.
It should also be possible to get an estimate for a currently running migration.

## M8: Migration scheduling

Add ability to schedule a migration.

- create the migration scheduling zone
- write queue handling for migrations
- connect with CNAPI for performing migrations
- update tooling (sdc-migrate, node-triton and portal) to support migration
  scheduling

# Open Questions and TODOs

## Backups

- do we want to create a permanent (or for a limited time) snapshot/backup of
  the migrated instance (e.g. stored in Manta) in case something goes amiss,
  i.e. it looks like migration was successful and the source instance was
  deleted, but in actual fact the target instance is not functioning and now we
  are SOL? An alternative to this is that we mark the original source instance
  as migrated and leave it around for later possible restoration, but this would
  not be good in the case of a CN being decommissioned.

## Images

- should the source instance image (and all of it's origin images) be available
  on the target CN? Ideally I would say yes, because some features (like
  CreateImageFromMachine) will depend on these images being available - but is
  this a hard requirement? I am thinking if we can get the image onto the target
  CN then do it, but if not it's not a deal breaker - maybe a warning should be
  issued in this case as some Triton features would no longer work.

- can KVM (and Bhyve?) skip the zfs send of the zone root dataset and just
  send the uuid-disk* datasets? Since the root dataset is created again by the
  target instance provisioning.

- can we optimize migration (and or zfs send) to make use of base images (i.e.
  to just perform a zfs incremental send from the base/source image)? c.f.
  [joyent/smartos-live#204](https://github.com/joyent/smartos-live/issues/204)
  If the target instance provisioning (reservation) also installs the source
  image to the target CN, then this should be able to work?

## Networking

- do the source and destination CNs need to be in the same subnet?

- are there any other restrictions imposed by instance and/or CN networking
  (e.g. arp, DNS, routing, firewall)?

- how to control the creation of networks such that two instances have the same
  IP address/MAC etc... does it initially provision the target instance without
  networks and then later remove these networks from the source and then re-add
  to the target instance?

## Other

- should migration compress the zfs send data?

- draw up an activity/flow diagram to show what happens (especially in case of
  failure) in each step of the migration process.

# Caveats

- CPU must to be the same on both source and target CN (e.g. 64-bit Intel
  with Vt/Vx support).
- Destination platform version needs to be equal to or greater than the source
  platform version.
- The image(s) from which the instance was created must still exist in IMGAPI
- Source/Dest time must be in sync.
- PCI-passthrough (and similar) devices will not be supported.
- Guest within a guest is not supported?
- Cannot migrate core Triton services?

# Tests

This is mostly a dev notes section to run tests on these items:

- delegated datasets
- multiple disks (e.g. KVM disk-0, disk-1, disk-2)
- SSH daemons (and their keys) check they work correctly in a migrated instance
- ensure cannot migrate the same instance at the same time (double migration)
- ensure other actions to the instance (start, stop, delete, etc...) cannot be
  performed while an instance is migrating
- ensure an owner cannot stop (abort) an operator controlled migration
- for incremental, ensure the instance has not been reprovisioned in the time
  between the initial snapshot send and the incremental snapshot send.
- failure cases, by the truckload

# References

- [Joyent Migration Docs](https://docs.joyent.com/private-cloud/instances/compute-nodes)
- [Legacy migrator](https://github.com/joyent/legacy-migrator/)
- [Someday-maybe push button migration](https://mo.joyent.com/docs/engdoc/master/roadmap/someday-maybe/push-button-migration.html)
- [vmbundle](https://github.com/joyent/smartos-live/blob/master/src/vmunbundle.c)
