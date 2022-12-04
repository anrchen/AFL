Does one input leading to more response time implies its mutation is going to get more response time?

[SpotFuzz](https://www.mdpi.com/2079-9292/10/24/3142/htm) has a good summary on how AFL works in detail.

How the **main loop** works:



1. read_testcases fn reads initial seeds
2. run_target fn tests target program, and record the byte map with trace_bits[]
3. perform_dry_run fn tests seeds 

    This is only run once for the initial seeds.

    1. calibrate_case fn

        This verifies whether a test case contains problems

4. cull_queue fn finds winners for previously-unseen bytes and marks them as favored.
5. fuzz_one fn to execute the fuzzing stuff. This triggers the feedback loop:

	

	cull_queue -> fuzz_one -> queue_cur -> trim_case, calculate_score, common_fuzz_stuff -> run_target, save_if_interesting -> add_to_queue -> calibrate_case -> cull_queue



How the **feedback progress** works:

common_fuzz_stuff -> run_target, save_if_interesting -> (common_fuzz_stuff | add_to_queue)



1. common_fuzz_stuff: mutation algorithms to mutate seeds and generate new tests.
2. run_target: run the target program with the tests.
    1. Track code coverage and report if we detect security violations
3. save_if_interesting: save interesting test cases. This function adds a test to the queue if a new edge is detected or hit counts changed.
    2. If nothing interesting happens, the execution returns to the main execution of common_fuzz_stuff: common_fuzz_stuff -> save_if_interesting -> common_fuzz_stuff
    3. otherwise, test the new test with calibrate_case:

    save_if_interesting -> add_to_queue, calibrate_case (seed selection algorithm)




How the **calibration** works:

calibrate_case -> init_forkserver, write_to_testcase, run_target, update_bitmap_score

The function calibrates new seeds and rearranges the seed queue.



1. write_to_testcase
2. run_target
3. has_new_bits
    1. This determines whether a newly mutated test case produces new paths.
4. update_bitmap_score
    2. This maintains a list of top_rated[] entries (initial score) for every byte in the bitmap.
    3. cull_queue


