This readme serves as a guide to users of this refactored AFL version.# Refactoring logs## afl-fuzz.c**detailed refactoring**

To simplify the afl process, we break down the approach into three core steps as follow:

- Environment setup
- Dry run
- Culling queue


### Step 1: Environment setup

The step contains some important variable initialization.
#### 1.1 on pf\_mode
`pf_mode` determines whether to enter performance fuzzing mode (line 139)
#### 1.2 on max\_bits
`max_bits` stores a new bitmap (line 158)
#### 1.3 on max\_counts
`max_counts` records the bits we haven't seen in performance fuzzing mode (line 160)
#### 1.4 on exec\_cksum\_pf
`exec_cksum_pf` records the checksum of the max counts trace (line 262)

#### 1.5 on read\_testcases()
This function detects test cases from `in_dir` (input directory with test cases), and places them in queue (with consistent ordering) `queued_at_start` with 1.6 `add_to_queue()`. 

Queue entries (in other words, test cases) with deterministic fuzzing are skipped in the process to save time. Deterministic fuzzing means that the generated mutations are always the same. Hence, they only need to be executed once per queue entry and never again.

At the end of this process, `queued_at_start` records the queued test cases.

#### 1.6 on add\_to\_queue()
This function append new test case to the queue. 

One thing to keep in mind is the struture of each `queue_entry` here (at line 248). The term `queue_entry` essentially refers to a test case. And the function also updates the `last_path_time` variable to take the current time.

```
struct queue_entry {

  u8* fname;                          /* File name for the test case      */
  u32 len;                            /* Input length                     */

  u8  cal_failed,                     /* Calibration failed?              */
      trim_done,                      /* Trimmed?                         */
      was_fuzzed,                     /* Had any fuzzing done yet?        */
      passed_det,                     /* Deterministic stages passed?     */
      has_new_cov,                    /* Triggers new coverage?           */
      var_behavior,                   /* Variable behavior?               */
      favored,                        /* Currently favored?               */
      fs_redundant;                   /* Marked as redundant in the fs?   */

  u32 bitmap_size,                    /* Number of bits set in bitmap     */
      exec_cksum,                     /* Checksum of the execution trace  */
      max_count_cksum;                /* Checksum of the max counts trace */

  u64 exec_us,                        /* Execution time (us)              */
      handicap,                       /* Number of queue cycles behind    */
      depth;                          /* Path depth                       */

  u8* trace_mini;                     /* Trace bytes, if kept             */
  u32 tc_ref;                         /* Trace bytes ref count            */

  struct queue_entry *next,           /* Next element, if any             */
                     *next_100;       /* 100 elements ahead               */

};
```

#### 1.7 on setup_shm()

The main feature of AFL is to use the instrumentation feedback from the target binary to assist to the generation of mutated inputs. Specifically, the instrumented target records the branch information during execution, which is later used by the fuzzer to determine the system flow and code coverage.To achieve this feature, AFL passes the `shared_mem[]` array (a shared memory) to the instrumented binary to capture the branch coverage. Then, the fuzzer uses the `trace_bits` variable to record the shared memory addresses. In this trace map, every byte set in the map can be thought of as a hit for a particular tuple `(branch_src, branch_dst)` found in the instrumented code.**1.7.1** <span style="color:blue">ref</span>: allocating shared memory

```
shm_id = shmget(IPC_PRIVATE, MAP_SIZE, IPC_CREAT | IPC_EXCL | 0600);
```

<span style="color:crimson">new</span>: allocating shared memory for MAX_SIZE

```
shm_id = shmget(IPC_PRIVATE, MAP_SIZE + (MAX_SIZE * sizeof(u32)), IPC_CREAT | IPC_EXCL | 0600);
```**1.7.2** <span style="color:blue">ref</span>: The implementation of this feature first starts in the `setup_shm()` function. We initialize `trace_bits` in the fuzzer with:```trace_bits = shmat(shm_id, NULL, 0);```	<span style="color:crimson">new</span>: Similarly, we initialize max_bits in the fuzzer.```if (max_count_mode) max_bits = (u32 *) (max_bits + MAP_SIZE)```
 
### Step 2: Dry run

This step performs a dry run on the test cases, and any errors are reported to the user. The procedure produces an initial queue and bitmap.

#### 2.1 on run_target()

<span style="color:crimson">todo for documentation</span>

**2.1** <span style="color:blue">ref</span>: After each target execution, the fuzzer clears up the `trace_bits` variable in the `run_target` function.
```memset(trace_bits, 0, MAP_SIZE); ```<span style="color:crimson">new</span>: Similarly, we reset max_bits.
```if (max_count_mode) memset(max_bits, 0, MAX_SIZE * sizeof(u32));```

#### 2.2 on calibrate_case()

Overall, this function starts up the fork server with `init_forkserver`, calibrates new test cases and invokes `update_bitmap_score` to give an initial score to the bytes. Particularly, it warns users about flaky or problematic test cases early on. It also uses a checksum to determine whether a newly mutated test case produces new paths. We then calculate the performance of each test case with `update_bitmap_score`.

**2.2** <span style="color:blue">ref</span>: AFL uses the `calibrate_case` function to verify whether a test case contain problems early on. While doing so, it also checks whether the newly mutated test case has introduced new execution paths (with `cksum`).
``` u32 cksum = hash32(trace_bits, MAP_SIZE, HASH_CONST);```<span style="color:crimson">new</span>: In our case, we want to perform the similar checksum whenever cksum is true, therefore, we can set our `max_count_cksum` variable in the else branch.
```if (max_count_mode)	q->max_count_cksum = hash32(max_bits, MAX_SIZE*sizeof(u32), HASH_CONST);```

#### 2.3 on update\_bitmap\_score()

We use this function to maintain a list of `top_rated[]` entries for every byte in the bitmap. It updates `top_rated` to record the new winner entries. See 3.1 for more information `top_rated`.

This function is called every time `calibrate_case()` is executed. 

**2.3** <span style="color:blue">ref</span>: `update_bitmap_score` function maintains the `top_rated` variable for each byte set in `trace_bits[]`.

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

#### 2.4 on save\_if\_interesting()

This function saves interesting input seeds into queue. The definition of "interesting" relies on whether the given test case visits a new tuple or the number of time visited changes (by using `has_new_bits`).

**2.4** <span style="color:blue">ref</span>: The initial file name takes the following format.

```
fn = alloc_printf("%s/queue/id:%06u,%s", out_dir, queued_paths, describe_op(hnb));
```

<span style="color:crimson">new</span>: We update the file name.

```
u8  hnm = 0;
if (max_count_mode)
	hnm = update_max_count()
...
fn = alloc_printf(..., describe_op(hnb), (max_ct_fuzzing && hnm) ? ",+max" : "" );
...
if (max_count_mode) 
	queue_top->max_count_cksum = hash32(max_bits, MAX_SIZE*sizeof(u32), HASH_CONST); 
```

#### 2.5 on perform\_dry\_run()

This function is essentially the driver function for the previously discussed sub-sections (2.1-2.3). At the end of the dry run, `check_map_coverage` function is called to verify the coverage map.

**2.5** <span style="color:blue">ref</span>: `perform_dry_run` function calls `check_map_coverage` if there is no fault (i.e., case `FAULT_NONE`).

```
perform_dry_run() {
...
}
```

<span style="color:crimson">new</span>: In performancefuzzing mode, we also need to configure `max_counts`.

```
perform_dry_run() {
	if (max_count_mode) update_max_count();
...
}
```

### Step 3: Culling queue
#### 3.1 on top\_rated

As we fuzz the program, the number of test cases in queue increases. While our goal is to hit every covered path, we do not need to execute all test cases. AFL applies a queue culling strategy to choose the optimal subset of test cases such that the test execution time and file size are minimized. The global variable `top_rated[MAP_SIZE]` is used to record the optimal test cases that is able to cover a particular path (i.e., based on the index of the array).

In particular, `top_rated[MAP_SIZE]` records a list of entries for every byte/tuple in the bitmap. This variable is mainly used by function `update_bitmap_score()`. For every byte set in `trace-bits[]`, if an entry has a more favorable (speed x size) factor, then it is marked as favored in `top_rated`.

<span style="color:crimson">check if this is true</span> A favored subset is then selected again by `cull_queue` function. The process sequentially marks previously-unseen bytes as favored, until the next run.**3.1** <span style="color:blue">ref</span>: `top_rated` declaration

```
static struct queue_entry* top_rated[MAP_SIZE];
```
<span style="color:crimson">new</span>: minor struct modification

```
static struct queue_entry** top_rated;
```

#### 3.2 on cull\_queue()

Again, the idea is to cover every path/tuple with the lowest-scoring test cases. At the end of this process, we obtain a queue of favored test cases that cover all tuples.

Particularly, the approach finds the next tuple not yet in the temporary working set (i.e., `temp_v`). From this tuple, it locates the winning queue entry (test case) by selecting lowest-scoring candidates (i.e., score proportional to its execution latency and file size). Then, it registers all tuples in that entry's trace (showed in `trace_mini`) in the working set and loops for missing tuples.

Here is an example of the workflow:

```
edge/byte: e0,e1,e2,e3,e4
test: t0,t1,t2
temp_v=[1,1,1,1,1]

t1 covers e2,e3, in other words [0,0,1,1,0]
t2 covers e0,e1,e4, in other words [1,1,0,0,1]

top_rated[0]=t2
top_rated[2]=t1

for loop at edge/byte 0
working set temp_v[0]=1, e0 is not covered
e0 is covered by t2, apply t2's coverage to trace_mini=[1,1,0,0,1]
update working set temp_v to remove the covered set, [0,0,1,1,0]
mark t2 as favored

for loop at edge/byte 1
working set temp_v[1]=0 as e1 is covered by t2
skip

for loop at edge/byte 2
working set temp_v[2]=1, e2 is not covered
e2 is covered by t1, apply t1's coverage to trace_mini=[0,0,1,1,0]
update working set temp_v to remove the covered set, [0,0,0,0,0]
mark t1 as favored

all edges have been covered, favored=t1,t2
```

Overall, this function loops through `top_rated`, and marks test case (i.e., queue entry) that triggers new edges (i.e., byte) as favored. It essentially chooses the best entries for each previously-unseen edge.

**3.2** <span style="color:blue">ref</span>: `cull_queue` function finds winners for previously-unseen bytes and marks them as favored.

Assumption: we find which tuple hasn't been covered from the test queue, we locate the winning test case, register all the tuples in that test, find more tuples.

```
static void cull_queue(void) {
...
}
```<span style="color:crimson">new</span>: We re-implement this in `max_count_mode`.

```
static void cull_queue(void) {

	/* performancefuzzing */
	if (max_count_mode){
		for (i = 0; i < MAX_SIZE; i++) {
		
		  if (top_rated[i]) {
		
		    /* if top rated for any i, will be favored */
		    u8 is_favored = top_rated[i]->favored;
		
		    top_rated[i]->favored = 1;
		
		    /* increments counts only if not also favored for another i */
		    if (!is_favored){
		
		      queued_favored++;
		
		      if (!top_rated[i]->was_fuzzed) pending_favored++;
		    }
		
		  }
		
		}
		return;
	}
	
...
}
```

#### 3.3 on trim\_case()

This function performs deletion on the test cases which should not have any impact on the trace.

<span style="color:blue">ref</span>: 

<span style="color:crimson>new</span>: 

```
u32 exec_cksum_pf;
if (pf_mode)
	exec_cksum_pf = hash32(max_bits, MAX_SIZE*sizeof(u32), HASH_CONST);
```
