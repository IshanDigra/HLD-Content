# System Design Comparison: Offset Pagination vs Cursor Pagination

---

## 1. The Decision Matrix (TL;DR)
| Feature | Offset Pagination | Cursor Pagination |
| :--- | :--- | :--- |
| **Primary Use Case** | Search results with explicit page numbers (e.g., E-commerce, Admin Tables). | Real-time streams and infinite scrolls (e.g., Social Media feeds). |
| **Mental Model** | **The Book Index:** "Skip the first 100 pages and show me the next 10." | **The Bookmark:** "Start exactly where I left off last time and show me the next 10." |
| **Key Strength** | **Navigability:** Supports jumping to specific pages (e.g., "Go to page 5"). | **Scale & Stability:** Constant-time performance and consistent results during writes. |
| **Key Weakness** | **Performance Decay:** Becomes exponentially slower as the offset increases. | **Limited Navigation:** Impossible to jump to a specific page or sort by non-unique fields easily. |

---

## 2. Core Mechanics & Architecture

* **Data Model & Storage:** * **Offset:** Uses a stateless "skip-and-take" approach via `LIMIT` and `OFFSET` SQL commands.
    * **Cursor:** Uses a stateful pointer (typically an indexed ID or timestamp) to locate the last seen record.
* **Consistency & Durability:** * **Offset:** Highly prone to **result inconsistency**. If a record is added or deleted on page 1 while a user is on page 2, the data shifts, causing duplicates or skipped records.
    * **Cursor:** Offers a **stable pagination window**. Since the query fetches records relative to a specific ID, insertions or deletions elsewhere in the dataset do not shift the current result set.
* **Scaling Strategy:** * **Offset:** Does not scale well with large datasets. The database must scan and count through all $(offset + limit)$ records before discarding the unwanted ones and returning the remaining results.
    * **Cursor:** Scales efficiently. By using an indexed column in the `WHERE` clause (e.g., `WHERE id <= %cursor`), the database "jumps" directly to the record without iterating through unwanted data.



---

## 3. Performance & Latency Profiles
* **Throughput:** **Cursor pagination** supports significantly higher throughput for deep-page queries because it avoids the "full scan" overhead of high offsets.
* **Latency:** **Offset pagination** latency increases drastically as the offset increases. **Cursor pagination** maintains near-constant latency regardless of depth, provided the cursor column is indexed.
* **Resource Overhead:** Offset pagination is "heavy" on the database engine for large datasets due to discarded row processing. Cursor pagination requires the client to store and send back the `next_cursor` state.

---

## 4. The Decision Tree (When to Choose What)
* **Pick Offset Pagination IF:**
  * The dataset is small or the UI specifically requires a "jump to page X" feature.
  * You need to show the total number of records or pages to the user.
* **Pick Cursor Pagination IF:**
  * You are dealing with large datasets that grow over time.
  * The data is highly dynamic with frequent additions/deletions.
  * The primary consumption model is "Infinite Scroll" or a mobile feed.

---

## 5. Senior-Level "Gotchas"
* **Offset Pitfalls:** **The "Deep Page" Killer.** As the offset increases, the query time increases drastically because the database must compute all preceding rows.
* **Cursor Pitfalls:** **Sorting & Uniqueness.** The cursor must come from a unique and sequential column (e.g., timestamp). If sorting by a non-unique column (like a first name), it becomes challenging to implement without potentially skipping data or creating complex, slower keys.



---

## 6. The "Interview Pivot" Script

> **The Choice:** "I’m choosing **Cursor Pagination** over **Offset** because our requirements prioritize **result consistency** and **low latency at scale** over the ability to jump to a specific page."
> 
> **Acknowledging the Trade-off:** "While Cursor Pagination introduces complexity regarding sorting by non-unique fields, we can mitigate this by concatenating columns to create a unique key, or by using an encoded cursor to provide a consistent interface."
> 
> **The Alternatives:** "If our scale was smaller or if we were building an internal admin tool where random access to pages is required, **Offset Pagination** would be the better 'off-the-shelf' fit."

---

## 7. Quick Summary Table
| Constraint | Winner | Why? |
| :--- | :--- | :--- |
| **Highest Throughput** | **Cursor** | Avoids scanning and discarding rows; utilizes index seeks. |
| **Lowest Latency** | **Cursor** | Performance remains stable even for "deep" pagination. |
| **Ease of Ops** | **Offset** | Standard SQL commands; no need to manage complex cursor states. |
| **Data Reliability** | **Cursor** | Immune to "drifting" pages caused by concurrent writes. |
| **Total Page Visibility**| **Offset** | Provides total records/pages, allowing clients to grasp the full volume. |

---

### Pro-Tip: Encoded Cursors
To maintain flexibility, return an **Encoded Base64 Cursor** regardless of the solution. When using offset pagination, encode the `page_number` and `total_pages`. When using cursor pagination, encode the `next_cursor`. This allows the server to change the underlying implementation without breaking the API contract for the client.