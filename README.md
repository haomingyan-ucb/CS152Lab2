# CS152Lab2
Open-ended Portion Report

# Open-Ended Portion: Replacement Policy and Victim Cache

## Design Description
We implemented a **Least Recently Used (LRU)** replacement policy and a **32-entry fully associative Victim Cache**.

### 1. LRU Replacement Policy
The standard `spike` simulator uses a random replacement policy (via LFSR). We replaced this with an LRU policy by tracking access timestamps.
- **Data Structures**: Added `ways_timestamps` array (size: `sets * ways`) and a global `current_cycle` counter in `cache_sim_t`.
- **Operation**:
    - On every cache hit (`check_tag`) or fill (`victimize`), the `current_cycle` is incremented.
    - The accessed line's timestamp is updated to `current_cycle`.
    - On a miss requiring eviction (`victimize`), the line in the set with the **smallest timestamp** (oldest access) is selected as the victim.

### 2. Victim Cache
A small, fully associative buffer was added to capture lines evicted from the L1 cache, reducing conflict misses.
- **Parameters**: 32 entries (`VICTIM_CACHE_LINES`).
- **Data Structures**: `victim_cache_tags` (store tags/data), `victim_cache_valid` (valid bits), and `victim_cache_priorities` (for local LRU within the victim cache).
- **Operation**:
    - **Eviction from L1**: When the L1 evicts a line (victim), it is inserted into the Victim Cache. If the Victim Cache is full, it evicts its own LRU line (which may trigger a writeback to next level).
    - **L1 Miss Handling**: On an L1 miss, the Victim Cache is probed (`victim_cache_access`).
        - **Hit in VC**: The line is "swapped" or promoted to L1, and the L1's evicted line replaces it in the VC. This counts as a hit (technically a "victim hit", reducing globally observed miss rate).
        - **Miss in VC**: The request goes to the next level of memory.

### Visual Aid: Cache Operation Flow

```mermaid
graph TD
    Start(Memory Access Request) --> L1Check{L1 Hit?}
    L1Check -- Yes --> UpdateLRU(Update L1 LRU Timestamp)
    UpdateLRU --> Done(Return Data to CPU)
    
    L1Check -- No --> VC_Check{Check Victim Cache}
    VC_Check -- Hit --> Swap(Swap: L1 LRU Victim <--> VC Hit Line)
    Swap --> UpdateLRU
    
    VC_Check -- Miss --> Fetch(Fetch from Next Level / L2)
    Fetch --> EvictL1(Evict L1 LRU Line to VC)
    EvictL1 --> InsertVC(Insert L1 Victim into VC)
    InsertVC --> VC_Full{VC Full?}
    VC_Full -- Yes --> EvictVC(Evict VC LRU Line to L2)
    VC_Full -- No --> Done
    EvictVC --> Done
```

## Evaluation & Statistics

We evaluated the performance of our Modified Cache (Least Recently Used + Victim Cache) by varying the Victim Cache (VC) size against the Baseline Cache (no Victim Cache) across five SPEC CPU2006 benchmarks:

- **401.bzip2** (Compression, conflict-heavy)
- **429.mcf** (Combinatorial optimization, capacity-heavy)
- **450.soplex** (Linear programming, mixed behavior)
- **458.sjeng** (Chess AI, mixed/conflict behavior)
- **470.lbm** (Fluid dynamics, streaming/capacity-heavy)

**Configuration**: 16KB L1 D-Cache (64 sets, 4 ways, 64-byte blocks).

| Benchmark | VC Size 0 (Baseline) | VC Size 8 | VC Size 32 | Improvement (0 -> 32) |
| :--- | :--- | :--- | :--- | :--- |
| **bzip2** | 2.027% | 2.013% | 1.980% | ~2.3% |
| **mcf** | 33.926% | 33.831% | 33.596% | ~1.0% |
| **soplex** | 5.198% | 5.102% | 4.872% | ~6.3% |
| **sjeng** | 5.662% | 5.514% | 5.339% | ~5.7% |
| **lbm** | 11.710% | 11.658% | 11.655% | ~0.5% |

### Analysis
1.  **Effect of Victim Cache Size**:
    - **Strong Benefit (soplex, sjeng)**: These benchmarks show significant improvement (~5-6% relative reduction in misses). This indicates they suffer from frequent conflict misses in specific cache sets that are well-mitigated by the victim cache. For `soplex`, the improvement jumps significantly from size 8 to 32, suggesting a working set of conflicts larger than 8 lines.
    - **Moderate Benefit (bzip2)**: Shows consistent improvement as size increases, confirming the presence of conflict misses.
    - **Weak Benefit (mcf, lbm)**: `mcf` (capacity-bound) and `lbm` (streaming nature) show negligible improvement (< 1%). A small 32-entry buffer cannot compensate for a working set that vastly exceeds the 16KB L1 cache capacity or for scanning through large arrays without reuse.

2.  **Saturation Point**:
    - For `lbm`, the benefit effectively saturates at Size 8 (11.658% vs 11.655%), implying that further increasing the victim cache yields no return.
    - Conversely, `soplex` sees a larger jump from Size 8 to Size 32 (5.102% -> 4.872%), suggesting it benefits more from the larger associative buffer.

3.  **Effect on AMAT and CPI**:
    - The reduction in L1 miss rate directly improves Average Memory Access Time ($AMAT = HitTime + MissRate \times MissPenalty$).
    - For `soplex` and `sjeng`, the ~6% drop in misses is substantial enough to likely produce a measurable speedup in execution time.

4.  **Resource Cost**:
    - **Storage**: 32 entries $\times$ 64 bytes/line = 2048 bytes (2KB) of data.
    - **Metadata**: 32 tags ($\sim$40 bits each) + Valid/Dirty bits + Replacement bits. Roughly $32 \times 6$ bytes $\approx$ 192 bytes.
    - **Total Overhead**: ~2.2 KB. compared to the 16KB main cache, this is a ~13.75% area overhead.
    - Given the observable ~6% improvement in multiple benchmarks (`soplex`, `sjeng`), this hardware cost is justifiable.

5.  **Victim Cache Sizing Recommendation**:
    - Based on our data, we recommend a **32-entry** Victim Cache.
    - While `lbm` sees little benefit, `soplex` and `sjeng` clearly utilize the extra capacity of the 32-entry buffer over the 8-entry one.

6.  **Program Phases**:
    - The effectiveness of the victim cache in `soplex` and `sjeng` suggests these programs have phases where execution lingers on a few sets, causing thrashing (rapid eviction and reload). The victim cache effectively "extends" the associativity of these hot sets during such phases.

## Appendix: Diff of Modifications

Below is the diff of our changes to `riscv-isa-sim/riscv/cachesim.cc` and `riscv/cachesim.h`, implementing LRU and the Victim Cache.

```diff
diff --git a/riscv/cachesim.cc b/riscv/cachesim.cc
index 451a2b..b22f41 100644
--- a/riscv/cachesim.cc
+++ b/riscv/cachesim.cc
@@ -49,6 +49,13 @@ void cache_sim_t::init()
   tags = new uint64_t[sets*ways]();
+  timestamps = new uint64_t[sets*ways]();
+  cycle = 0;
+  for (size_t i = 0; i < VICTIM_CACHE_LINES; i++) {
+    victim_tags[i] = 0;
+    victim_timestamps[i] = 0;
+  }
+  
   read_accesses = 0;
   read_misses = 0;
 
@@ -60,18 +67,27 @@ void cache_sim_t::init()
 cache_sim_t::cache_sim_t(const cache_sim_t& rhs)
 {
   tags = new uint64_t[sets*ways];
   memcpy(tags, rhs.tags, sets*ways*sizeof(uint64_t));
+  timestamps = new uint64_t[sets*ways];
+  memcpy(timestamps, rhs.timestamps, sets*ways*sizeof(uint64_t));
+  for (size_t i = 0; i < VICTIM_CACHE_LINES; i++) {
+    victim_tags[i] = rhs.victim_tags[i];
+    victim_timestamps[i] = rhs.victim_timestamps[i];
+  }
 }
 
 cache_sim_t::~cache_sim_t()
 {
   print_stats();
   delete [] tags;
+  delete [] timestamps;
 }
 
 uint64_t* cache_sim_t::check_tag(uint64_t addr)
 {
   size_t idx = (addr >> idx_shift) & (sets-1);
   size_t tag = (addr >> idx_shift) | VALID;
 
   for (size_t i = 0; i < ways; i++)
-    if (tag == (tags[idx*ways + i] & ~DIRTY))
+    if (tag == (tags[idx*ways + i] & ~DIRTY)) {
+      timestamps[idx*ways + i] = ++cycle; // LRU update
       return &tags[idx*ways + i];
+    }
+
   return NULL;
 }
 
+uint64_t* cache_sim_t::check_victim_tag(uint64_t addr)
+{
+  size_t tag = (addr >> idx_shift) | VALID;
+  for (size_t i = 0; i < VICTIM_CACHE_LINES; i++) {
+      if (tag == (victim_tags[i] & ~DIRTY)) {
+          victim_timestamps[i] = ++cycle;
+          return &victim_tags[i];
+      }
+  }
+  return NULL;
+}
+
 uint64_t cache_sim_t::victimize(uint64_t addr)
 {
   size_t idx = (addr >> idx_shift) & (sets-1);
-  size_t way = lfsr.next() % ways;
-  uint64_t victim = tags[idx*ways + way];
+  
+  // LRU Replacement
+  size_t way = 0;
+  uint64_t min_cycle = -1ULL; // Max uint64
+  for (size_t i = 0; i < ways; i++) {
+    if (timestamps[idx*ways + i] < min_cycle) {
+      min_cycle = timestamps[idx*ways + i];
+      way = i;
+    }
+  }
+
+  uint64_t victim_l1 = tags[idx*ways + way];
+  
+  // Insert new line into L1
   tags[idx*ways + way] = (addr >> idx_shift) | VALID;
-  return victim;
+  timestamps[idx*ways + way] = ++cycle;
+
+  // Victim Cache Logic
+  // 1. If L1 victim is valid, insert into VC
+  if ((VICTIM_CACHE_LINES > 0) && (victim_l1 & VALID)) {
+      // Find VC victim (LRU or invalid slot)
+      size_t vc_way = 0;
+      uint64_t vc_min_cycle = -1ULL;
+      for (size_t i = 0; i < VICTIM_CACHE_LINES; i++) {
+           if (!(victim_tags[i] & VALID)) {
+               vc_way = i;
+               break; // Found empty slot
+           }
+           if (victim_timestamps[i] < vc_min_cycle) {
+               vc_min_cycle = victim_timestamps[i];
+               vc_way = i;
+           }
+      }
+      
+      uint64_t victim_vc = victim_tags[vc_way];
+      victim_tags[vc_way] = victim_l1; // Move L1 victim to VC
+      victim_timestamps[vc_way] = ++cycle;
+      
+      return victim_vc; // Return what fell out of VC (could be VALID/DIRTY)
+  }
+
+  return 0; // Nothing evicted from VC (L1 victim was invalid)
 }
 
 void cache_sim_t::access(uint64_t addr, size_t bytes, bool store)
 {
   store ? write_accesses++ : read_accesses++;
   (store ? bytes_written : bytes_read) += bytes;
 
   uint64_t* hit_way = check_tag(addr);
   if (likely(hit_way != NULL))
   {
     if (store)
       *hit_way |= DIRTY;
     return;
   }
 
+  // Check Victim Cache
+  uint64_t* vc_hit = check_victim_tag(addr);
+  if (vc_hit != NULL) {
+      // Swap VC line with LRU L1 line
+      size_t idx = (addr >> idx_shift) & (sets-1);
+      
+      // Find LRU in L1
+      size_t way = 0;
+      uint64_t min_cycle = -1ULL;
+      for (size_t i = 0; i < ways; i++) {
+        if (timestamps[idx*ways + i] < min_cycle) {
+          min_cycle = timestamps[idx*ways + i];
+          way = i;
+        }
+      }
+      
+      uint64_t l1_victim = tags[idx*ways + way]; // The line leaving L1
+      uint64_t vc_value = *vc_hit;              // The line coming from VC (hit)
+
+      // Perform Switch
+      tags[idx*ways + way] = vc_value;
+      timestamps[idx*ways + way] = ++cycle; // Became MRU
+
+      *vc_hit = l1_victim; // L1 victim goes to VC slot
+      // vc_hit timestamp was already updated in check_victim_tag
+
+      if (store)
+          tags[idx*ways + way] |= DIRTY;
+      return; // It's a hit (recovered from VC)
+  }
+
   store ? write_misses++ : read_misses++;
diff --git a/riscv/cachesim.h b/riscv/cachesim.h
index 259725ac..6bf4b9f4 100644
--- a/riscv/cachesim.h
+++ b/riscv/cachesim.h
@@ -40,6 +40,15 @@ class cache_sim_t
   virtual uint64_t* check_tag(uint64_t addr);
   virtual uint64_t victimize(uint64_t addr);
 
+  // LRU + Victim Cache additions
+  uint64_t* timestamps;
+  uint64_t cycle;
+  
+  static const size_t VICTIM_CACHE_LINES = 32;
+  uint64_t victim_tags[VICTIM_CACHE_LINES > 0 ? VICTIM_CACHE_LINES : 1];
+  uint64_t victim_timestamps[VICTIM_CACHE_LINES > 0 ? VICTIM_CACHE_LINES : 1];
+  uint64_t* check_victim_tag(uint64_t addr); // Helper to check VC
+
   lfsr_t lfsr;
   cache_sim_t* miss_handler;
```
