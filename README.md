# Assignment 1: Performance Analysis on a Quad-Core CPU

## Program 1: Parallel Fractal Generation Using Threads (20 points)

最简单直观的实现：按照高度依次平均分配给每个线程 ，每个线程的totalRows= height/ numThreads,加速比比较低

| 线程数 | view1加速比 | view2加速比 |
| ------ | ----------- | ----------- |
| 2      | 1.98        | 1.65        |
| 3      | 1.66        | 2.16        |
| 4      | 2.42        | 2.50        |
| 5      | 2.41        | 2.82        |
| 6      | 3.12        | 3.07        |
| 7      | 3.27        | 3.56        |
| 8      | 3.72        | 3.78        |

原因：每行的亮度不一样，循环次数有多有少，总的运行时间取决于计算量最大的线程

![image-20230627151524538](images\image-20230627151524538.png)

解决方案：每个线程以线程数量作为行间隔计算， 即i+=numTreads

<img src="images\image-20230627151841401.png" alt="image-20230627151841401" style="zoom:50%;" />

可以看到加速比能到达7.19x,但是开16个线程的效果跟8个线程差不多，因为还是不能将计算量平均分配到这么多个线程中，有些线程计算量很大，有些线程计算量很小。

## Program 2: Vectorizing Code Using SIMD Intrinsics (20 points)

Program 2大体上就是体验以下SIMD编程，根据实际的代码翻译以下就好了。需要注意的问题是当N%VECTOR_WIDTH不等于0时，需要额外处理

```c
      maskAll = _cs149_init_ones(min(VECTOR_WIDTH,N-i));   //最后一组数据不足向量宽度时，只需要实际大小即可
//后续的mask需要和maskAll与以下，即只要实际大小的，否则计算会出错，会越界，或者陷入死循环
      maskRemain= _cs149_mask_and(maskRemain,maskAll);


```



| 向量宽度 | 利用率 |
| -------- | ------ |
| 2        | 85.3%  |
| 4        | 82.1%  |
| 8        | 77.3%  |
| 16       | 77.7%  |
| 32       | 76.8%  |
| 64       | 76.9%  |
| 128      | 76.8%  |

## Program 3: Parallel Fractal Generation Using ISPC (20 points) ##



###  Part 1. A Few ISPC Basics 

What is the maximum speedup you expect given what you know about these CPUs? Why might the number you observe be less than this ideal? 

Answer: Ideally,the maximum speedup might be 8x  since  the ISPC compiler emit **8-wide** AVX2 vector instructions. But I only achieve 5.17x  speedup for view 1 and 4.34x for view 2.  The reason is straightforward like the situation in Program 1: the computation is unbalanced.

### Part 2: ISPC Tasks 

1. Run `mandelbrot_ispc` with the parameter `--tasks`. What speedup do you
     observe on view 1? What is the speedup over the version of `mandelbrot_ispc` that
     does not partition that computation into tasks?

   answer: With the parameter --tasks,the speedup can be up to 9.67x on view 1 while the speedup is only 5.06x without that.

2. There is a simple way to improve the performance of
     `mandelbrot_ispc --tasks` by changing the number of tasks the code
     creates. By only changing code in the function
     `mandelbrot_ispc_withtasks()`, you should be able to achieve
     performance that exceeds the sequential version of the code by over 32 times!
     How did you determine how many tasks to create? Why does the
     number you chose work best?

   answer: I determine 16 tasks to create and the speedup is up to 29.92x. 

   ![image-20230627093346182](images\image-20230627093346182.png)

3. _Extra Credit: (2 points)_ What are differences between the thread
     abstraction (used in Program 1) and the ISPC task abstraction? There
     are some obvious differences in semantics between the (create/join
     and (launch/sync) mechanisms, but the implications of these differences
     are more subtle. Here's a thought experiment to guide your answer: what
     happens when you launch 10,000 ISPC tasks? What happens when you launch
     10,000 threads? (For this thought experiment, please discuss in the general case

### Program 4: Iterative `sqrt`

1. Build and run `sqrt`. Report the ISPC implementation speedup for 
     single CPU core (no tasks) and when using all cores (with tasks). What 
     is the speedup due to SIMD parallelization? What is the speedup due to 
     multi-core parallelization?

   The version of SIMD parallelization can achieve 4.25x speedup.The speedup is up to 29.83x due to multi-core parallelization.

2. Modify the contents of the array values to improve the relative speedup 
     of the ISPC implementations. Construct a specifc input that __maximizes speedup over the sequential version of the code__ and report the resulting speedup achieved (for both the with- and without-tasks ISPC implementations). Does your modification improve SIMD speedup?
     Does it improve multi-core speedup (i.e., the benefit of moving from ISPC without-tasks to ISPC with tasks)? Please explain why.

   I construct an array with all elements equal to 2.998f. Since all the elements are the same, each lane requires the same calculation time, which leads to no-mask in the SIMD parallelization. I achieve 6.08x speedup in ISPC,and 38.33x speedup in task ISPC.

3. Construct a specific input for `sqrt` that __minimizes speedup for ISPC (without-tasks) over the sequential version of the code__. Describe this input, describe why you chose it, and report the resulting relative performance of the ISPC implementations. What is the reason for the loss in efficiency? 
   __(keep in mind we are using the `--target=avx2` option for ISPC, which generates 8-wide SIMD instructions)__. 

   I construct an array with all elements equal to 1.f. The speedup of ISPC is 2.85x, and the speedup of task ISPC is 3.28x.

4. _Extra Credit: (up to 2 points)_ Write your own version of the `sqrt` 
    function manually using AVX2 intrinsics. To get credit your 
   implementation should be nearly as fast (or faster) than the binary 
   produced using ISPC. You may find the [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/) 
   very helpful.





### Program 5: BLAS `saxpy`

1. Compile and run `saxpy`. The program will report the performance of
     ISPC (without tasks) and ISPC (with tasks) implementations of saxpy. What 
     speedup from using ISPC with tasks do you observe? Explain the performance of this program.
     Do you think it can be substantially improved? (For example, could you rewrite the code to achieve near linear speedup? Yes or No? Please justify your answer.)

   The speedup  with task ISPC is only 0.92x. It might be memory bandwidth limiting the speedup.

2. __Extra Credit:__ (1 point) Note that the total memory bandwidth consumed computation in `main.cpp` is `TOTAL_BYTES = 4 * N * sizeof(float);`.  Even though `saxpy` loads one element from X, one element from Y, and writes one element to `result` the multiplier by 4 is correct.  Why is this the case? (Hint, think about how CPU caches work.)

3. __Extra Credit:__ (points handled on a case-by-case basis) Improve the performance of `saxpy`.
   We're looking for a significant speedup here, not just a few percentage 
     points. If successful, describe how you did it and what a best-possible implementation on these systems might achieve. Also, if successful, come tell the staff, we'll be interested. ;-)
