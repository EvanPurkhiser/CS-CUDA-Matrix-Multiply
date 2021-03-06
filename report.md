# CUDA Matrix Multiplication

Evan Purkhiser

In this lab I will be calculating the product of two matrixes using the NVIDIA
CUDA toolkit. I will be using the 8 block 8x8 thread variation of the matrix
multiplication program for my trials.

I will be performing a total of four experiments. Half of the experiments will
be performed on the University of Akron 'Tesla' server, which uses a NVIDIA
C2050 Tesla card. The other half of the experiments will be performed on one of
the Computer Science lab machines which utilize a NVIDIA GTX 480 consumer
graphics card. As a bonus I will also do some small comparisons to my desktop
machine that utilizes a NVIDIA GTX 760 card. On these machines I will be
recording the time trials of running the program without blocked matrix
multiplication (as is), and then using blocked matrix multiplication.

## Linux

Tun run the program on the Tesla server as well as on my Linux desktop a small
change had to be made to the program itself. The Makefile to build the program
also needed to be changed slightly to account for variations in the installation
path of the CUDA toolkit.

First, since we are using CUDA 5 we must changed the `cutil.h` include to
`helper_cuda.h`.

To handle compiling the program on Linux I created a small Makefile that will
download the required data files and compile the program using a specified
`CUDAPATH`.

```makefile
CUDAPATH=/opt/cuda

dnsmb1: dnsmb1.cu product.txt matrix1.txt matrix2.txt
	$(CUDAPATH)/bin/nvcc -I $(CUDAPATH)/samples/common/inc/ dnsmb1.cu -o dnsmb1 -arch=sm_20

%.txt:
	wget http://www.cs.uakron.edu/~toneil/cs477/Labs/lab5/$@
```

Note that I've included a `CUDAPATH` variable that indicates the location of the
CUDA toolkit installation directory. On my machines CUDA is installed into
`/opt/cuda` while on the tesla server it is installed to `/usr/local/cuda`. This
allows us to easily change the CUDA path for build time.

I've also added a build target to download the required testing data files
(matrix{1,2}.txt and product.txt).

## Experiment 1

For the first experiment I ran the program and recorded execution times without
making any changes.

I ran the program with the command `for r in {1..10}; do ./dnsmb1; done`

| Trial 1      | Trial 2      | Trial 3      | Trial 4      | Trial 5      | Trial 6      | Trial 7      | Trial 8      | Trial 9      | Trial 10     |
| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------|
| $285.952\mu$ | $281.440\mu$ | $281.440\mu$ | $284.832\mu$ | $284.288\mu$ | $279.296\mu$ | $285.120\mu$ | $287.008\mu$ | $285.664\mu$ | $286.656\mu$ |

| Minimum time | Maximum time | Average Time | STD      |
| ------------ | ------------ | ------------ | -------- |
| $279.269\mu$ | $287.008\mu$ | $284.200\mu$ | $2.575$  |

And just for fun here is some bonus data from this same program being run on my
desktop Linux machine.

| Minimum time | Maximum time | Average Time | STD        |
| ------------ | ------------ | ------------ | ---------- |
| $114.112\mu$ | $495.808\mu$ | $189.300\mu$ | $136.4\mu$ |

### Discussion

The primary thing we can take note of here is just how consistent the times were
on the Tesla server. With only a standard deviation of $2.5\mu s$ we can see
that the CUDA program doesn't have a lot of play in execution time.

Just for fun I decided to let the tesla server do some more work and executed
the command `time while true; do ./dnsmb1; done > times.txt` and let it run for
a little while. The results from this showed again a standard deviation of just
$3.292$. That's out of a total of 123 trials. (The `time` failed to report to
standard error for some reason however).

Comparing this to my desktop Linux machine we can see there was actually a much
higher standard deviation of the 10 trials, with the maximum time being more
than 4 times that of the minim execution time. I speculate that this could be in
part due to the difference in the "consumer" grade graphics cards versus the
"server" grade devices.

Something else to note that wasn't shown in the results however was that the
startup time for the program was actually much faster on my desktop machine than
it was on the tesla server. On my desktop executing `time ./dnsmb1` gives me a
real time of $0.050s$, while on the tesla server it actually takes $2.277s$. I
did a quick investigation into this by using the `strace` utility with the -tt
option to display the time stamp (with microseconds) for each system call. I was
able to clearly see that the tesla server hung on the `open("/dev/nvidia0",
O_RDWR)` call, returning a `EINTR` indicating that the device was actually busy.

## Experiment 2

For the second experiment we will be converting the CUDA Matrix
Multiplication program over to using a Blocked Matrix Multiplication algorithm
and calculating the product of two 16x16 matrices.

This was actually quite a bit trickier than first anticipated. As shown in the
product description we would need to break the matrix up into four tiles, each
tile has two matrix products that must be calculated and the added together,
giving us a total of 8 extra groups to do work.

The approach I took to getting this working is as follows:

 1. Add a second dimension to the block grid of size 8. This dimension will be
	our 'group id' which will be used to determine which tile offset we are calculating.

 2. Determine which tile we are calculating in the device function by looking at
	our group ID and calculating rows and column offsets.

 3. Add the calculated product to into the same product index. For this
	opeartion we must use the `atomicAdd` CUDA function since we are reading and
	writing global memory.

Another important change I was required to make was to ensure that the memory
for the product was zeroed out on the device.

Again, I ran the program with the command `for r in {1..10}; do ./dnsmb1; done`.

\pagebreak

| Trial 1      | Trial 2      | Trial 3      | Trial 4      | Trial 5      | Trial 6      | Trial 7      | Trial 8      | Trial 9      | Trial 10     |
| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------|
| $282.720\mu$ | $284.576\mu$ | $282.752\mu$ | $288.608\mu$ | $284.992\mu$ | $290.656\mu$ | $287.360\mu$ | $285.088\mu$ | $292.384\mu$ | $287.520\mu$ |

| Minimum time | Maximum time | Average Time | STD      |
| ------------ | ------------ | ------------ | -------- |
| $282.720\mu$ | $292.387\mu$ | $286.700\mu$ | $3.234$  |

And again, just for fun, here is some extra bonus data from executing the program
on my desktop machine

| Minimum time | Maximum time | Average Time | STD        |
| ------------ | ------------ | ------------ | ---------- |
| $147.424\mu$ | $208.416\mu$ | $197.400\mu$ | $18.16\mu$ |

### Discussion

The most obvious thing we can notice here is that there doesn't seem to actually
be any sort of improvement in speed. In fact, on average this version of the
program actually runs $2\mu$ seconds slower than the 8x8 non-tiled program.
Though, we should remember that we are actually calculating a matrix that is a
power of two larger than the first experiment.

Again we can see that the standard deviation of times on the tesla server is
much lower than that of the times on my desktop card. This time however, we can
notice that the standard deviation of my desktop machine was much lower. To
confirm this I ran the program over 1000 times and calculated the standard
deviation from the reported times. With over 1000 trials I saw a standard
deviation of $31.840$, which is slightly larger than the previously calculated
STD, but nowhere near as big of a STD as shown in the first experiment.

# Windows

For the next two experiments I will be using one of the University of Akron lab
machines that include a NVIDIA GTX 480 CUDA capable graphics device. For the
setup we can use the same two programs we used on Linux (including the change
for `helper_cuda.h`). The primary difference here is how we build the program.

A `winrun.bat` batch file is included to compile and run the program, however I
did have to make a small change to the environment `PATH` variable to include
the Visual Studio 10.0 CL compiler.

    set PATH=%PATH%;C:\Program Files\Microsoft Visual Studio 10.0\VC\bin

I completely discarded the Makefile used on my Linux machine.

I did spend more time than I would like to admit attempting to debug why the
program would not work on the 'older' windows machines in the lab before
realizing that I needed to use the 'newer' machines. On the older machines
things would compile fine, however the computed matrix would always be filled
with zeros.

## Experiment 1

I ran the program ten separate times and recorded the execution times.

The results are as follows:


| Trial 1      | Trial 2      | Trial 3      | Trial 4      | Trial 5      | Trial 6      | Trial 7      | Trial 8      | Trial 9      | Trial 10     |
| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------|
| $280.969\mu$ | $321.888\mu$ | $280.640\mu$ | $313.728\mu$ | $392.800\mu$ | $279.040\mu$ | $280.352\mu$ | $309.472\mu$ | $281.920\mu$ | $280.544\mu$ |

| Minimum time | Maximum time | Average Time | STD        |
| ------------ | ------------ | ------------ | ---------- |
| $279.040\mu$ | $392.800\mu$ | $302.100\mu$ | $35.89\mu$ |

### Discussion

Interestingly enough the results were similar to how my Linux desktop machine
behaved in terms of variations of times. We have a standard deviation of $35\mu
s$, which is certainly quite a large jump from the Tesla machine's minuscule
$2.575\mu s$. We can also note that the times between the lab machines GTX 480
consumer grade graphics cards were rather comparable to the Tesla machines
server grade device.

## Experiment 2

I will be using the same program used in the Linux Experiment 2.

| Trial 1      | Trial 2      | Trial 3      | Trial 4      | Trial 5      | Trial 6      | Trial 7      | Trial 8      | Trial 9      | Trial 10     |
| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------| -------------|
| $290.848\mu$ | $291.520\mu$ | $291.968\mu$ | $291.968\mu$ | $335.136\mu$ | $318.720\mu$ | $325.824\mu$ | $453.920\mu$ | $289.760\mu$ | $283.904\mu$ |

| Minimum time | Maximum time | Average Time | STD        |
| ------------ | ------------ | ------------ | ---------- |
| $283.904\mu$ | $453.920\mu$ | $317.400\mu$ | $16.19\mu$ |

### Discussion

We can see that our results are actually slightly slower than running the
programs on the tesla server. Also, just as in the Linux experiment, we can see
that our times are actually only a fraction slower than the 8x8 times in the
first experiment done on windows. Again, the standard deviation seems to be a
little more consistent compared to the first experiment.

