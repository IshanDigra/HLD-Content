# Count-Min Sketch (CMS): System Design Notes

## 1. The "Elevator Pitch"
In a system design interview, if you are asked to design a system that tracks the frequency of millions or billions of events (e.g., "design a system to find the top 10 trending hashtags on Twitter" or "detect a DDoS attack by counting requests per IP"), keeping an exact count of every single item using a Hash Map will run out of memory. 

**Count-Min Sketch (CMS)** is a probabilistic data structure that serves as a **frequency table for massive data streams**. It trades perfect accuracy for extreme memory efficiency. 
* **The Catch:** It may *overestimate* the count of an item due to hash collisions, but it will **never underestimate** it.

## 2. Core Structure & Mechanics
A Count-Min Sketch is surprisingly simple. It consists of two main components:
1. **A 2D Array (Matrix):** Let's say it has `d` rows and `w` columns. All cells are initialized to 0.
2. **Hash Functions:** You have `d` independent hash functions, one for each row. 

### A. Insertion (Writing)
When a new item arrives in the stream (e.g., the word "apple"):
1. Pass "apple" through all `d` hash functions.
2. Each hash function gives you an index for its specific row (modulo `w` so it fits in the columns).
3. Go to that cell in each row and **increment the counter by 1**.
*(Result: For 1 item, you update exactly `d` cells).*

### B. Querying (Reading)
To find out how many times "apple" has been seen:
1. Pass "apple" through the same `d` hash functions to find its specific cells.
2. Look at the counts in those `d` cells.
3. **Return the Minimum value** among them. 

**Why the Minimum?** Because other items might have collided with "apple" in some of the rows, artificially inflating those specific counters. The cell with the *lowest* value has suffered the fewest collisions and is closest to the true count. 

## 3. The Math: Space vs. Accuracy Trade-off
In an interview, you must explain how to size the sketch. The dimensions (`w` and `d`) directly control your error rate.
* **Width (`w`):** Controls the *magnitude* of the error. Wider matrix = fewer collisions. `w = ceil(e / epsilon)`
* **Depth (`d`):** Controls the *probability* of the error. More rows = higher confidence in the result. `d = ceil(ln(1 / delta))`

*Example:* To guarantee that your error is within 0.1% with 99.9% certainty, you can calculate exact, fixed dimensions. The memory footprint remains constant regardless of whether you process 1 million or 1 billion events!

## 4. Top System Design Use Cases
When should you bring up CMS in an interview?
* **Heavy Hitters (Top K Problem):** Combine a CMS with a Min-Heap. As items come in, update the CMS, query the new estimated count, and if it's larger than the smallest item in your Min-Heap of size K, insert it.
* **Trending Topics:** Tracking the frequency of search queries on Google or hashtags on social media.
* **Network Security / Rate Limiting:** Counting packets coming from specific IP addresses to detect DDoS attacks in real-time routers with limited memory.
* **Database Query Planning:** Databases use sketches to estimate the frequency of values in a column to optimize `JOIN` operations.

## 5. Pros and Cons

| Advantages | Disadvantages |
| :--- | :--- |
| **Fixed Memory:** `O(w * d)` space, independent of the stream size. | **Overestimation:** It is an approximate counter. |
| **Fast:** `O(d)` time complexity for both inserts and lookups. | **No Deletions:** You cannot easily decrement a count, because that cell might be shared with another item (unless you use a more complex variant). |
| **Mergeable:** You can easily add two sketches together (if they have the same dimensions and hash functions) to combine data from distributed servers. | **Requires good Hash Functions:** Poorly distributed hash functions will ruin the accuracy. |
