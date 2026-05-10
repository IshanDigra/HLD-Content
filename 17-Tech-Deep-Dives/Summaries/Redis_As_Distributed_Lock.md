# System Design: Redis as a Distributed Lock

## 1. The Real-World Problem

In a system like **BookMyShow**, thousands of users might click on the exact same seat (e.g., `Seat_A10`) at the exact same millisecond.

If the system handles these requests in parallel without coordination:

- User A and User B both see the seat as "Available"
- Both proceed to payment
- Both pay money, but only one can actually have the seat

**Result:** A **double booking disaster** and a customer support nightmare

---

## 2. The "Post-it Note" Mental Model

Think of Redis as a shared, lightning-fast **bulletin board** that all your application servers can see.

- When a user selects a seat, the server tries to stick a Post-it note:
    - `"Seat_A10 is currently being booked by User_123"`
- If a note already exists:
    - Other servers reject the request → "Seat is temporarily locked"
- If no note exists:
    - The server places its note and proceeds

---

## 3. Technical Mechanics (Interview Script)

### A. Acquiring the Lock (SET NX EX)

Instead of "check then set" (which causes race conditions), Redis performs this **atomically**:

```bash
SET lock:seat:A10 user_token NX EX 300
```

- **NX (Not Exists):** Only creates the key if it doesn’t already exist
- **EX (Expire):** Sets TTL (e.g., 5–10 minutes)
    - Prevents deadlocks if user drops mid-payment

---

### B. Releasing the Lock

After payment success or cancellation:

```bash
DEL lock:seat:A10
```

This unlocks the seat for others.

---

## 4. Why Redis over SQL?

- **High Performance**
    - Handles 100k+ requests/sec
    - In-memory → sub-millisecond latency

- **Decoupling**
    - Avoids polluting primary DB with temporary locks
    - Keeps booking DB clean and transactional

---

## 5. Senior-Level Edge Cases

### What if Redis crashes? (Redlock)

- Single Redis node failure → all locks lost

**Solution: Redlock**
- Use 5 independent Redis nodes
- Lock is valid only if **3 out of 5 nodes succeed (quorum)**

---

### What if a process freezes? (Fencing Tokens)

Scenario:

- Server acquires lock for 5 minutes
- Freezes for 6 minutes
- Lock expires
- Another user acquires lock
- Old server resumes and tries to complete booking

**Solution: Fencing Tokens**
- Each lock gets a version number (1, 2, 3...)
- DB only accepts writes with the **latest token**
- Prevents stale writes from old processes

---

## 6. Summary Cheat Sheet

- **Best for:** Preventing duplicate work (seat booking, payments)
- **Key Command:** `SET NX EX`

### Pros
- Sub-millisecond latency
- Easy to implement
- Highly scalable

### Cons
- Not perfectly consistent in all failure scenarios
- For strict financial guarantees → use SQL transactions or systems like ZooKeeper