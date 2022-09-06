This readme serves as a guide to users of this refactored AFL version.Refactoring logsafl-fuzz.c

**general notes**line 139: max_count_mode    determines whether to enter max_count_modeline 158: perf_bits -> max_bits    stores a new bitmapline 160: max_counts    records the bits we haven't seen in max_count_modeline 262: max_count_cksum    record the checksum of the max counts trace**detailed refactoring**
**1. on max_bits**
The main feature of AFL is to use the instrumentation feedback from the target binary to assist to the generation of mutated inputs. Specifically, the instrumented target records the branch information during execution, which is later used by the fuzzer to determine the system flow and code coverage.To achieve this feature, AFL passes the `shared_mem[]` array (a shared memory) to the instrumented binary to capture the branch coverage. Then, the fuzzer uses the trace_bits variable to record the shared memory addresses. In this trace map, every byte set in the map can be thought of as a hit for a particular tuple (branch_src, branch_dst) found in the instrumented code.**1.1** <span style="color:blue">ref</span>:The implementation of this feature first starts in the `setup_shm()` function. We initialize `trace_bits` in the fuzzer with:```trace_bits = shmat(shm_id, NULL, 0);```	<span style="color:crimson">new</span>:
Similarly, we initialize max_bits in the fuzzer.```if (max_count_mode) max_bits = (u32 *) (max_bits + MAP_SIZE)```**1.2** <span style="color:blue">ref</span>: After each target execution, the fuzzer clears up the trace_bits variable in the run_target function.
```memset(trace_bits, 0, MAP_SIZE); ```<span style="color:crimson">new</span>: Similarly, we reset max_bits.
```if (max_count_mode) memset(max_bits, 0, MAX_SIZE * sizeof(u32));```**1.3** <span style="color:blue">ref</span>: AFL uses the calibrate_case function to verify whether a test case contain problems early on. While doing so, it also checks with cksum (checksum) whether the mutated inputs has introdcued new execution paths.
``` u32 cksum = hash32(trace_bits, MAP_SIZE, HASH_CONST);```<span style="color:crimson">new</span>: In our case, we want to perform the similar checksum whenever cksum is true, therefore, we can set our `max_count_cksum` variable in the else branch.
```if (max_count_mode)	q->max_count_cksum = hash32(max_bits, MAX_SIZE*sizeof(u32), HASH_CONST);```

**2. on top_rated**

As we fuzz the program, the number of test cases in queue increases. While our goal is to hit every covered path, we do not need to execute all test cases. AFL applies a queue culling strategy to choose the optimal subset of test cases such that the test execution time and file size are minimized. The global variable `top_rated[MAP_SIZE]` is used to record the optimal test cases that is able to cover a particular path (i.e., based on the index of the array).

In particular, `top_rated[MAP_SIZE]` records a list of entries for every byte in the bitmap. This variable is mainly used by function `update_bitmap_score()`. For every byte set in `trace-bits[]`, if an entry has a more favorable (speed x size) factor, then it wins a slot in `top_rated`. A favored subset is then selected again by `cull_queue` function. The process sequentially marks previously-unseen bytes as favored, until the next run.**2.1** <span style="color:blue">ref</span>: `top_rated` declaration

```
static struct queue_entry* top_rated[MAP_SIZE];
```
<span style="color:crimson">new</span>: minor struct modification

```
static struct queue_entry** top_rated;
```

**2.2** <span style="color:blue">ref</span>: `update_bitmap_score` function maintains the `top_rated` variable for each byte set in `trace_bits[]`.

```
static void update_bitmap_score(struct queue_entry* q) {
...
}
```

<span style="color:crimson">new</span>: We re-implement the same idea in `max_count_mode` to boost the entry with max count.

```
static void update_bitmap_score(struct queue_entry* q) {

  /* performancefuzzing */
  if (max_count_mode){

    /* Insert if we achieve the max count */ 
    for (i = 0; i < MAX_SIZE; i++)

      if (max_bits[i]) {

         if (top_rated[i] && (max_bits[i] < max_counts[i])) continue;

         /* Insert ourselves as the new winner. */
         top_rated[i] = q;
         score_changed = 1;

      }

    return;
  }
  
...
}
```