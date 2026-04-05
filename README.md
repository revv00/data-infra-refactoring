# Solve Syncing Hell with a Shared Folder for $0.20/year ($6 after the 1st year)

> *Sometimes you don't need petabyte scale. You just need three computers to agree on what is in a folder — without you having to write the code to make them agree.*

---

## Table of Contents

- [Background](#background)
- [The Setup](#the-setup)
- [The Problem: Syncing Hell](#the-problem-syncing-hell)
- [Candidate Solutions](#candidate-solutions)
- [The Solution: JuiceFS](#the-solution-juicefs)
  - [Cost Breakdown](#cost-breakdown)
  - [Setup](#setup)
  - [Bandwidth Optimization](#bandwidth-optimization)
- [The Killer Feature: Death of Sync Logic](#the-killer-feature-death-of-sync-logic)
- [Dictionary Management](#dictionary-management)
- [Pitfalls & Realities](#pitfalls--realities)

---

## Background

Software maintenance is often defined by a specific flavor of anxiety — that low-level dread you feel when managing moving parts you aren't particularly interested in, but which are vital to your work.

This writeup documents a full refactor of the data layer across a personal multi-machine ML stack. It started with a hardware scare and ended with the realization that **"syncing" is a problem we should no longer be solving manually.**

---

## The Setup
<img width="608" height="590" alt="infra0" src="https://github.com/user-attachments/assets/70f5c717-34fd-41a2-bce3-e17317a4e951" />

The algorithm system relies on four distinct machines, each with a specific role:

| Box | Hardware | Role |
|-----|----------|------|
| **IBOX** (Ingestion) | Tiny VPS — 2GB RAM / 2-core | Entry point. Runs Python scrapers + hosts Elasticsearch (single source of truth) |
| **PBOX** (Production) | RTX 2070 Super | Daily inference. Turned on/off daily to save power |
| **RBOX** (Research) | RTX 3090 / 128GB RAM | ML experiments (CNN, GCN, RL) and Spark jobs |
| **EBOX** (Execution) | MacBook Pro (Intel / 16GB) | Control center. Aging but still running |

### The Inciting Incident

Both GPUs have been running for 7 and 5 years. When PBOX refused to boot, it triggered a frantic drive to the physical machine — which turned out to be a false alarm. But the damage was done mentally.

The real question surfaced: ***"Could I actually reconstruct these pipelines if this machine died for real?"***

After starting to dockerize everything, skeletons fell out of the closet: inconsistent dictionaries, missing files, fragmented data. The real fix wasn't Docker — it was the **Data Infra**.

---

## The Problem: Syncing Hell

In the legacy setup, each box synced from Elasticsearch at different frequencies:

- **PBOX:** Daily syncs
- **RBOX:** Multi-monthly syncs
- **EBOX:** Ad-hoc, infrequent syncs for visualization

On the surface, this worked. The devil was in the implementation.

### The "Devils" of Manual Syncing

1. **Triple the work** — A flaw in upstream data meant fixing downstream logic in at least three different scripts.
2. **Maintenance debt** — A bug found in the RBOX script required auditing all the others.
3. **Complexity** — 10GB is "medium" data. Too small to justify full downloads every time, too large to be cavalier about it.

---

## Candidate Solutions

| Candidate | Verdict |
|-----------|---------|
| **GDrive / OneDrive** | Zero maintenance, but failed on close-to-open consistency |
| **Amazon EFS** | Great consistency, but EC2-only — non-starter for local boxes |
| **Azure Files** | NFS requires the "Premium" tier — overkill for this budget |
| **CephFS / GlusterFS** | Too much Ops work. Here to write algos, not manage clusters |
| **JuiceFS** | POSIX-compatible, uses object storage, surprising free tier |
| **PGFS** | Interesting (JuiceFS + PostgreSQL), but requires managing a DB for both blob and meta |

---

## The Solution: JuiceFS

JuiceFS had been on the radar for years. This time, the cost and setup were scrutinized properly.

### Cost Breakdown

| Component | Estimated | Actual |
|-----------|-----------|--------|
| Object Store (50GB) | ~$6/year | **~$0.20** (first year trial) |
| JuiceFS Metadata | ~$12/year | **$0** (free up to 1TB + 1M inodes) |
| Bandwidth | Variable | **$0** (colocated object store + proxy) |

**Total: ~$0.20 for year one. $6 or so afterwards**

### Setup

Run the following on each box (assuming object store and metadata are already provisioned):

```bash
# Authenticate
juicefs auth "$VOL_NAME" --token "$TOKEN" \
  --accesskey "$ACCESS_KEY" --secretkey "$SECRET_KEY"

# Mount
juicefs mount -d "$VOL_NAME" \
  --cache-dir "$CACHE_DIR" \
  --cache-size "$CACHE_SIZE" \
  "$MOUNT_DIR"
```

### Bandwidth Optimization

To eliminate object store egress fees, route traffic through a proxy VPS in the same region:

```bash
# On clients (not the proxy itself)
export HTTP_PROXY="$PROXY_URL" HTTPS_PROXY="$PROXY_URL"

juicefs auth "$VOL_NAME" --token "$TOKEN" \
  --accesskey "$ACCESS_KEY" --secretkey "$SECRET_KEY"

juicefs mount -d "$VOL_NAME" \
  --cache-dir "$CACHE_DIR" \
  --cache-size "$CACHE_SIZE" \
  "$MOUNT_DIR"
```

> **Note:** You'll still need to run a Squid proxy on the VPS. It's a one-time setup, but it is an extra component to track.

### Result
<img width="608" height="590" alt="infra1" src="https://github.com/user-attachments/assets/300ef140-e025-463d-ac1a-cafab43e2661" />

By mounting a JuiceFS shared folder as the single source of truth across all three boxes — functionally equivalent to what the Elasticsearch index previously provided — three separate syncing scripts collapsed into one. No more incremental sync handling. No more hunting across systems for bugs or data corruption.

---

## The Killer Feature: Death of Sync Logic

The most underrated part of JuiceFS is the **local read cache**. It sounds minor. It is not.

**Old world:** Write "Sync Logic" — code to decide what to fetch, what to keep, and how to handle deltas. With multiple data sources, this effort multiplies.

**New world:** Point scripts at `/mnt/jfs/data` and let the filesystem figure it out.

### How the Cache Works

| Read | Behavior |
|------|----------|
| First read | File is fetched from the Object Store and cached locally |
| Subsequent reads | Served at local disk speed |

No more "Logic A" for daily runs and "Logic B" for monthly research. The cognitive load of managing data transfers evaporated entirely.

---

## Dictionary Management

Dictionaries (mapping files, versioned lookups) are a nightmare for ML projects. They're atomic but change as the project evolves. Previously synced via `scp` or `rsync`, which led to **version drift** — losing track of which box had which version.

The shared folder solves this cleanly:

- No more SSH keys just to sync files
- No more `rsync`
- No more frequency calculus ("Is this dictionary current?")
- Write and read directly from the shared folder
- One version exists, or you rename it

---

## Pitfalls & Realities

| Issue | Detail |
|-------|--------|
| **File layout matters** | JuiceFS handles small-file appends (e.g., 10,000 tiny daily appends) poorly. Prefer writing new immutable files. |
| **The Squid proxy** | Required to eliminate egress fees. One-time setup, but still a component to maintain. |
| **Backups still matter** | Reliability compounds until it doesn't. A weekly automated backup of the JuiceFS folder to an external drive remains in place. |
| **Windows (non-POSIX)** | Windows guest VMs can't mount JuiceFS natively without issues. **WinFsp** resolves this cleanly — no need for a third mounting approach. |

---

## Key Takeaway

JuiceFS markets itself for AI and Big Data. For the solo developer or researcher managing a multi-box setup, its greatest value is simpler: **a reliable, consistent, shared filesystem with a brilliant local cache.**
