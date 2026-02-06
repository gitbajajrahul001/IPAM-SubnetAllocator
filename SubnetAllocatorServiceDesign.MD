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

These are facts, not assumptions.

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

## Step 3: Re-CIDR the free ranges (this is the missing step)

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

## Step 4: Coalesce adjacent free blocks (important)

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
