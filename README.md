<p align="center">
  <h1 align="center"> ChampSim For CS422A </h1>
  <p> ChampSim is a trace-based simulator for a microarchitecture study. You can sign up to the public mailing list by sending an empty mail to champsim+subscribe@googlegroups.com. A set of traces used for the 2nd Cache Replacement Championship (CRC-2) can be found from this link. (http://bit.ly/2t2nkUj) <p>
</p>

# Steps to integrate DRAMSim2 with ChampSim
1. Clone the ChampSim repository.
```
git clone https://github.com/Prakharj24/ChamSim-CS422A.git
```
2. Go inside ChamSim directory and clone the DRAMSim2 repository.
```
git clone https://github.com/umd-memsys/DRAMSim2.git
```

3. Inside DRAMSim2 directory, rename the file system.ini.example to system.ini .
```
mv system.ini.example system.ini
```

# Compile

ChampSim takes five parameters: Branch predictor, L1D prefetcher, L2C prefetcher, L2C replacement policy, LLC replacement policy, and the number of cores. 
For example, `./build_champsim.sh bimodal no no lru 1` builds a single-core processor with bimodal branch predictor, no L1/L2 data prefetchers, and the baseline LRU replacement policy for the LLC.
```
$ ./build_champsim.sh bimodal no no lru 1

$ ./build_champsim.sh ${BRANCH} ${L1D_PREFETCHER} ${L2C_PREFETCHER} ${LLC_REPLACEMENT} ${NUM_CORE}
```

# Run simulation

Copy `scripts/run_champsim.sh` to the ChampSim root directory and change `TRACE_DIR` in `run_champsim.sh` <br>

* Single-core simulation: Run simulation with `run_champsim.sh` script.

```
$ ./run_champsim.sh bimodal-no-no-lru-lru-1core 1 10 bzip2_183B

$ ./run_champsim.sh ${binary} ${n_warm} ${n_sim} ${trace} ${option}

${binary}: ChampSim binary compiled by "build_champsim.sh" (bimodal-no-no-lru-1core)
${n_warm}: number of instructions for warmup (1 million)
${n_sim}:  number of instructinos for detailed simulation (10 million)
${trace}: trace name (bzip2)
${option}: extra option for "-low_bandwidth" (src/main.cc)
```
Simulation results will be stored under "results_${n_sim}M" as a form of "${trace}-${binary}-${option}.txt".<br> 

* Multi-core simulation: Run simulation with `run_4core.sh` or `run_8core.sh`. <br>
Note that `${trace}` is replaced with `${num}` that represents a unique ID for randomly mixed multi-programmed workloads. 

```
$ ./run_4core.sh ${binary} ${n_warm} ${n_sim} ${num} ${option}

${num}: mix number is the corresponding line number written in sim_list/4core_workloads.txt
```

# Add your own branch predictor, data prefetchers, and replacement policy
**Copy an empty template**
```
$ cp branch/branch_predictor.cc prefetcher/mybranch.bpred
$ cp prefetcher/l1d_prefetcher.cc prefetcher/mypref.l1d_pref
$ cp prefetcher/l2c_prefetcher.cc prefetcher/mypref.l2c_pref
$ cp replacement/l2c_replacement.cc replacement/myrepl.l2c_repl
$ cp replacement/llc_replacement.cc replacement/myrepl.llc_repl
```

**Work on your algorithms with your favorite text editor**
```
$ vim branch/mybranch.bpred
$ vim prefetcher/mypref.l1d_pref
$ vim prefetcher/mypref.l2c_pref
$ vim replacement/myrepl.l2c_repl
$ vim replacement/myrepl.llc_repl
```

**Compile and test**
```
$ ./build_champsim.sh mybranch mypref mypref myrepl 1
$ ./run_champsim.sh mybranch-mypref-mypref-myrepl-myrepl-1core 1 10 bzip2_183B
```
# Analyze the workloads

The lru replacement policy for LLC and L2C allows the analysis of the workloads that it is
running. The features supported are:-

* **Tracing**: Allows for collection of access trace of the cache which can be further used for
  offline analysis. The trace is saved in a binary file with following structure:

```
  struct access {
    uint64_t full_addr;
    uint32_t type;
  }
```
* **Reuse Distance**: Calculates the number of distinct access between the access to the same block.
    It only considers load and store requests while calculating the reuse distance as writeback requests
    are not part of the program characteristics and are noise introduced by the hardware.

* **Access and Misses**: Calculates the blocks per access i.e. number of blocks having 1 access, 2 access and so on.
    Similarly for the misses also.

The LLC supports the following features:-
* TRACE_LLC
* REUSE_LLC
* ACCESS_COUNT_LLC
* MISSES_COUNT_LLC

The last 3 features are calculated separately for user and kernel instructions. The results are written to files that are
hardcoded in lru.llc_repl file.

L2C supports the following:-
* TRACE_L2C
* REUSE_L2C

L2C does not differentiate between user and kernel instructions.

**Activating a Feature**

To activate a feature we need to add "-DFEATURE_NAME" to the `CFlags` in the `Makefile`. For e.g.

```
CFlags = -Wall -O3 -std=c++11 -DTRACE_LLC
```

**Belady's OPT Algorithm**

The scripts folder also contains a implementation of the OPT algorithm which takes as input access trace files
which are generated by the TRACE feature described above. It prints the misses encountered by the OPT algorithm.

# How to create traces

We have included only 4 sample traces, taken from SPEC CPU 2006. These 
traces are short (10 million instructions), and do not necessarily cover the range of behaviors your 
replacement algorithm will likely see in the full competition trace list (not
included).  We STRONGLY recommend creating your own traces, covering
a wide variety of program types and behaviors.

The included Pin Tool champsim_tracer.cpp can be used to generate new traces.
We used Pin 3.2 (pin-3.2-81205-gcc-linux), and it may require 
installing libdwarf.so, libelf.so, or other libraries, if you do not already 
have them. Please refer to the Pin documentation (https://software.intel.com/sites/landingpage/pintool/docs/81205/Pin/html/)
for working with Pin 3.2.


**Use the Pin tool like this**
```
pin -t obj-intel64/champsim_tracer.so -- <your program here>
```

The tracer has three options you can set:
```
-o
Specify the output file for your trace.
The default is default_trace.champsim

-s <number>
Specify the number of instructions to skip in the program before tracing begins.
The default value is 0.

-t <number>
The number of instructions to trace, after -s instructions have been skipped.
The default value is 1,000,000.
```
For example, you could trace 200,000 instructions of the program ls, after
skipping the first 100,000 instructions, with this command:
```
pin -t obj/champsim_tracer.so -o traces/ls_trace.champsim -s 100000 -t 200000 -- ls
```
Traces created with the champsim_tracer.so are approximately 64 bytes per instruction,
but they generally compress down to less than a byte per instruction using xz compression.

# Evaluate Simulation

ChampSim measures the IPC (Instruction Per Cycle) value as a performance metric. <br>
There are some other useful metrics printed out at the end of simulation. <br>

Good luck and be a champion! <br>
