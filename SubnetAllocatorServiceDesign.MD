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
```
---
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


