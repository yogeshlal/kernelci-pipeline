---
title: "kernelci-lab Setup (External Scheduler on Maestro Events)"
date: 2026-03-27
description: "How to run a local scheduler that consumes production pipeline events"
weight: 4
---

This document describes the **kernelci-lab setup**, where a local `kernelci-pipeline` scheduler subscribes to production events from the KernelCI API and triggers jobs when incoming events match scheduler rules.

With this setup, you can:

- Run the `kernelci-pipeline` scheduler locally
- Subscribe to production events as an external user
- Filter events by:
  - `owner=production`
  - `submitter=service:pipeline`
- Trigger LAVA jobs (for example, `baseline-arm64-qualcomm`) from matching build events

## What This Setup Does

Using local checkouts of `kernelci-core` and `kernelci-pipeline`, the scheduler listens for node events from the KernelCI API and evaluates them against the rules defined in [`config/scheduler.yaml`](../config/scheduler.yaml).

When a matching `kbuild` event is received, the scheduler:

1. Matches the event against the scheduler rules
2. Creates the corresponding job nodes
3. Submits LAVA jobs using the configured runtime

For the Qualcomm `kernelci-lab` setup, the typical entries are:

- Runtime: `lava-kci-qualcomm` (from [`config/pipeline.yaml`](../config/pipeline.yaml))
- Typical scheduler target: `baseline-arm64-qualcomm`
- Event source: `kbuild-gcc-14-arm64-node-event`

## Prerequisites

Before running the scheduler, ensure that the following are available:

1. Local checkouts of:
   - `kernelci-pipeline`
   - `kernelci-core`

2. A valid apt_config and storage_confg configured in [`config/kernelci.toml`](../config/kernelci.toml):

```toml
api_config = "production"
storage_config = "kci-storage-production"
```

3. A valid KernelCI API token configured in [`config/kernelci.toml`](../config/kernelci.toml):

```toml
[storage.kci-storage-production]
storage_cred = "?sv=REPLACE_WITH_API_TOKEN"
```

4. A LAVA runtime token configured in [`config/kernelci.toml`](../config/kernelci.toml):

```toml
[runtime.lava-kci-qualcomm]
runtime_token="REPLACE_WITH_LAVA_RUNTIME_TOKEN"
```

## Scheduler CLI Options Used in This Setup

The scheduler was extended with options needed for external subscriptions:

- `--promisc`
  - Subscribe in promiscuous mode (receive events from all users visible to token)
- `--event-owner`
  - Filter events by top-level `owner`
- `--event-submitter`
  - Filter events by top-level `submitter`
- `--disable-device-health-check` (optional)
  - Do not skip jobs if no online devices are reported by LAVA
- `--disable-watchdog` (optional)
  - Disable watchdog force-exit for stuck scheduler threads


## Run Locally

From `kernelci-pipeline`:

```bash
export KCI_API_TOKEN="REPLACE_WITH_API_TOKEN"
export PYTHONPATH="${PYTHONPATH}:kernelci-core"

python3 ./src/scheduler.py \
  --settings=./config/kernelci.toml \
  loop \
  --runtimes lava-kci-qualcomm \
  --name production \
  --promisc \
  --event-owner production \
  --event-submitter service:pipeline \
  --disable-device-health-check \
  --disable-watchdog
```

If you want strict default behavior, remove optional flags:

- remove `--disable-device-health-check`
- remove `--disable-watchdog`


### Confirm source events exist

From `kernelci-core`:

```bash
python kci node find \
  kind='kbuild' \
  name='kbuild-gcc-14-arm64' \
  owner='production' \
  submitter='service:pipeline' \
  created__gte='2026-03-26T00:00:00'
```

### Confirm child jobs are created

For a given kbuild ID:

```bash
python kci node find kind='job' name='baseline-arm64-qualcomm' parent='<KBUILD_NODE_ID>'
```

## Troubleshooting

### `has no online devices`

Meaning:
- Scheduler asked LAVA for devices with `online_only=True`
- No `Good` (or equivalent allowed) device was found for that device type

Action:
- Use `--disable-device-health-check` to continue submission even when offline
- Verify LAVA device health values for the affected `device_type`

### Watchdog force exit

If enabled, watchdog can terminate process when worker threads appear stuck.
To keep scheduler alive during deep debugging:

- run with `--disable-watchdog`
