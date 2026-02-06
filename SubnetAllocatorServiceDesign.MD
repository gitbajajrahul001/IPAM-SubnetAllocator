# Concrete, Auditable IPAM Setup - Conceptual Walkthrough with an Example

## Supernet (single site)

**Supernet:**  
10.20.0.0/22

**Address range:**  
10.20.0.0 → 10.20.3.255

---

## Existing allocations (already in SolarWinds)

Let’s say IPAM currently has:

- 10.20.1.0/26  
- 10.20.1.64/27  
- 10.20.3.0/24  

---

## Step 1: Convert everything into address ranges

| CIDR      | Start        | End            |
|-----------|--------------|----------------|
| Supernet  | 10.20.0.0    | 10.20.3.255    |
| /26       | 10.20.1.0    | 10.20.1.63     |
| /27       | 10.20.1.64   | 10.20.1.95     |
| /24       | 10.20.3.0    | 10.20.3.255    |

---

## Step 2: Subtract allocated ranges from the supernet

What’s left numerically:

- 10.20.0.0 – 10.20.0.255  
- 10.20.1.96 – 10.20.1.255  
- 10.20.2.0 – 10.20.2.255  

No CIDR logic yet — just raw gaps.

---

## Step 3: Re-CIDR the free ranges

Now we convert each gap into the largest possible aligned CIDR blocks.

### Gap 1  
10.20.0.0 – 10.20.0.255  
→ 10.20.0.0/24

---

### Gap 2  
10.20.1.96 – 10.20.1.255  

Break it down:

- 96–127 → /27  
- 128–255 → /25  

So:

- 10.20.1.96/27  
- 10.20.1.128/25  

---

### Gap 3  
10.20.2.0 – 10.20.2.255  
→ 10.20.2.0/24

---

## Step 4: Coalesce adjacent free blocks

Now check adjacency + alignment:

- 10.20.2.0/24  
- 10.20.3.0/24 is NOT free → cannot merge  

So no merge here.

---

## Final free block map

- 10.20.0.0/24  
- 10.20.1.96/27  
- 10.20.1.128/25  
- 10.20.2.0/24  

# Next Step: Score Eligible Free Blocks for the Requested Subnet Size

At this stage, we **do not allocate anything**.  
We only decide **which free block is safest to touch**.

Input data is the **final free block map**, unchanged.

---

## Clean free-block map (input)

- 10.20.0.0/24  
- 10.20.1.96/27  
- 10.20.1.128/25  
- 10.20.2.0/24  

---

## Incoming request

**Requested subnet size:** `/27`

---

## Step 1: Filter eligible free blocks

Rule: block.prefix ≤ requested_prefix


Meaning: the block must be **same size or larger** than the requested subnet.

| Free block        | Prefix | Eligible for /27 |
|-------------------|--------|------------------|
| 10.20.0.0/24      | /24    | Yes              |
| 10.20.1.96/27     | /27    | Yes              |
| 10.20.1.128/25    | /25    | Yes              |
| 10.20.2.0/24      | /24    | Yes              |

All blocks are technically eligible.

---

## Step 2: Calculate fragmentation cost per block

We now score **impact**, not availability.

---

### Block A  
**10.20.1.96/27**

- Exact size match
- Remaining space after carve: **none**
- No fragmentation introduced
- No effect on future allocations

**Fragmentation cost:**  
Very low

**Reusability impact:**  
None

**Strategic value destroyed:**  
None

---

### Block B  
**10.20.1.128/25**

- /25 carved into one /27
- Remaining space: `/27` + `/26`
- Still cleanly aligned
- Still reusable for multiple sizes

**Fragmentation cost:**  
Low

**Reusability impact:**  
Minimal

**Strategic value destroyed:**  
Moderate (breaks a /25)

---

### Block C  
**10.20.0.0/24**

- /24 carved into one /27
- Remaining space: multiple fragments
- Destroys a full /24 option
- Reduces ability to place future /24 or larger workloads

**Fragmentation cost:**  
High

**Reusability impact:**  
Medium

**Strategic value destroyed:**  
High

---

### Block D  
**10.20.2.0/24**

- Same characteristics as Block C
- One of the largest remaining blocks in the site
- Critical for future scale

**Fragmentation cost:**  
Very high

**Reusability impact:**  
High

**Strategic value destroyed:**  
Very high

---

## Step 3: Rank blocks (best-fit + future safety)

Ranking principles:

- Prefer the **smallest block that fits**
- Penalize use of **/24 or larger blocks**
- Preserve **future growth options**

| Rank | Block           | Reason |
|-----:|-----------------|--------|
| 1    | 10.20.1.96/27   | Exact fit, zero fragmentation |
| 2    | 10.20.1.128/25  | Acceptable, controlled impact |
| 3    | 10.20.0.0/24    | Avoid unless necessary |
| 4    | 10.20.2.0/24    | Do not touch if possible |

---

## Result of this step

- **No IP allocated**
- **No subnet carved**
- A **preferred block order** is established

You now know **which block is safest to touch**  before making any allocation decision.

---

# Next Step: Select the Exact Child CIDR from the Chosen Free Block

At this point:

- The free-block map is built  
- Blocks are scored  
- One parent free block is selected as the safest  

Still **no allocation is committed**.

Now we decide **which exact subnet inside the chosen block** would be used.

---

## Given

**Chosen free block:**  
10.20.1.128/25

**Requested subnet size:**  
/27

---

## Step 1: Subdivide the parent block

A `/25` subdivides cleanly into **4 × /27** blocks:

- 10.20.1.128/27  
- 10.20.1.160/27  
- 10.20.1.192/27  
- 10.20.1.224/27  

These are all mathematically valid children.

---

## Step 2: Apply the edge-only rule

You may choose **only edge subnets**.

Valid choices:

- **Low edge:**  
  10.20.1.128/27  

- **High edge:**  
  10.20.1.224/27  

Invalid choices (never select):

- 10.20.1.160/27  
- 10.20.1.192/27  

Middle allocation is forbidden.

---

## Step 3: Pick one edge consistently

Define a policy **once** and never change it:

- Always lowest-address  
**or**
- Always highest-address  

Example (lowest-address policy):

Chosen child CIDR:  
10.20.1.128/27

---

## Why this step exists (important)
 
Allocating from the middle creates fragmentation: [ free ][ allocated ][ free ]


This produces **two fragments**, both smaller and harder to reuse.

Edge allocation produces: [ allocated ][ free ]


Effects:

- Single remaining fragment
- Future merges remain possible
- Fragmentation is predictable and controlled
- Long-term IPAM health is preserved

---

## Result of this step

- Exact child CIDR is **selected**
- Nothing is written to SolarWinds
- No allocation is committed

The next step after this is **atomic commit + conflict handling**.

# Next Step: Atomic Commit with Optimistic Locking

At this point you already have:

- Site  
- Requested prefix  
- Chosen parent free block  
- Exact child CIDR to allocate  

Now you must **safely commit** the allocation into SolarWinds.

SolarWinds provides **no native locking**, so correctness must be enforced by the allocator.

---

## What you do (strict order)

---

## 1️⃣ Re-read authoritative state

Immediately before creation:

- Re-fetch **all allocated subnets** for the site from SolarWinds
- Recompute the **free-block map**
- Re-validate that the **chosen child CIDR is still free**

Decision:

- If the CIDR is still free → continue  
- If the CIDR is no longer free → **abort and restart from the scoring step**

No partial progress is kept.

This is the optimistic-lock boundary.

---

## 2️⃣ Create subnet in SolarWinds

Call the SolarWinds IPAM API to:

- Create the subnet
- Associate it with the correct site
- Store metadata:
  - Request ID
  - Timestamp
  - Requester

This API call is the **commit point**.

If this call succeeds, the allocator assumes ownership.

---

## 3️⃣ Verify creation

Immediately after creation:

- Confirm the subnet exists
- Confirm the CIDR matches **exactly**
- Confirm **no overlap** was introduced

Verification is mandatory.

If **any check fails**:

- Treat the operation as failed
- Abort
- Retry from Step 1

Never assume success based on API response alone.

---

## 4️⃣ Bounded retry logic

Retry the entire sequence only **N times** (for example, `N = 3`):

- Re-read
- Re-score
- Re-select
- Re-commit

If all retries fail:

- Exit cleanly
- Return error: **“capacity contention”**

This prevents:

- Infinite retry loops
- Race-condition storms
- Cascading failures under load

---

## Why this step matters

SolarWinds IPAM has:

- No locking
- No reservation semantics
- No transactional guarantees

So **your allocator becomes the lock**.

This exact pattern is used in:

- Cloud IPAM systems
- Distributed schedulers
- Storage allocators

Anywhere shared state exists without native concurrency control.

---

## Where you are now (checkpoint)

You have designed:

- A fragmentation-aware allocator  
- Safe deletion and reuse  
- Concurrency control  
- Deterministic behavior  

At this point, you are no longer **calling IPAM APIs**.

You are **building IPAM logic**.

---

# Reality Check (important)

An end-to-end, production-safe IP allocator is **not** a “single script”.

It is a **small service** with clearly separated responsibilities.

Trying to collapse this into one file guarantees:
- Hidden bugs
- Race conditions
- Unrecoverable fragmentation
- Fear-driven “don’t touch IPAM” behavior

---

## What a real allocator actually contains

A production-grade allocator has **explicit layers**:

- **CIDR math engine**  
  All address arithmetic, splitting, merging, alignment checks

- **Allocation policy**  
  Fragmentation-aware logic  
  Best-fit selection  
  Edge-only child selection  

- **IPAM adapter**  
  SolarWinds API integration  
  No business logic  
  No CIDR decisions  

- **Concurrency + idempotency guard**  
  Optimistic locking  
  Re-reads  
  Conflict detection  

- **Retry + safety rails**  
  Bounded retries  
  Clean failure modes  
  Deterministic behavior  

This is not optional. This is table stakes.

---

## What comes next

You will be given:

- A **reusable allocator service design**
- A **complete, runnable reference implementation** in:

  - **Python** (recommended for this use case)
  - **PowerShell** (fully possible, slightly more verbose)

Both implementations will be:

- DRY  
- Idempotent  
- Safe to re-run  
- Optimistic-lock aware  
- Production-structured (not a toy script)

---

## Deployment flexibility

Either implementation can later be wrapped as:

- REST API  
- Azure Function  
- GitOps pipeline step  
- Self-service portal backend  

The allocator logic does **not** change.

Only the entrypoint does.

---

## Architecture (same for Python & PowerShell)

allocator/
├── ipam_client (SolarWinds adapter)
├── cidr_engine (math, split, merge)
├── allocator_policy (best-fit + edge allocation)
├── allocator_service (orchestration)
└── entrypoint (CLI / API)


---

## Key takeaway

If your allocator logic lives:
- inside API calls
- inside UI workflows
- inside “one big script”

…you don’t have an allocator.

You have a **time bomb with logging**.

What you’re building here is **infrastructure logic** —  
the same class of system as schedulers, storage allocators, and cloud IPAM engines.





