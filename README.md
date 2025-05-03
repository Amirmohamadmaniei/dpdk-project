# dpdk-project


# Project Report – [Special Topics in Computer Network 3]

## Team Members
- 👤 AmirMohamad Maniei
- 👤 Shayan Moradi
- 👤 AliReza Biria

---

## Tools & Technologies
- Linux
- DPDK
- Trace Compass
- Lttng
- ftrace
- perf


---
### Intallation of DPDK
- sudo apt update
- sudo apt install -y meson ninja-build python3-pyelftools build-essential
- git clone https://github.com/DPDK/dpdk.git
- cd dpdk
- meson setup build
- cd build
- ninja

---

### HugePages
- echo 1024 | sudo tee /proc/sys/vm/nr_hugepages
- cat /proc/meminfo | grep HugePages
- photo

### helloworld
- cd dpdk/<build_dir>
- meson configure -Dexamples=helloworld
- ninja
- ./<build_dir>/examples/dpdk-helloworld -l 0-3 -n 4
- photo

---

### memif
- ./<build_dir>/app/dpdk-testpmd -l 0-1 --proc-type=primary --file-prefix=pmd1 --vdev=net_memif,role=server -- -i

- ./<build_dir>/app/dpdk-testpmd -l 2-3 --proc-type=primary --file-prefix=pmd2 --vdev=net_memif -- -i

 **Client :**
 - testpmd> start

**Server :**
- testpmd> start tx_first

**both :**
- testpmd> show port stats 0

 photo
 photo

---

### lttng function tracing
- lttng create session
- lttng enable-event --userspace --all
- lttng add-context --userspace --type=vpid --type=vtid --type=procname  
- lttng start
* sudo LD_PRELOAD=/usr/lib/x86_64-linux-gnu/liblttng-ust-cyg-profile.so ./app/dpdk-testpmd -l 0-1 --proc-type=primary --file-prefix=pmd1 --vdev=net_memif,role=server -- -i
- sudo LD_PRELOAD=/usr/lib/x86_64-linux-gnu/liblttng-ust-cyg-profile.so ./app/dpdk-testpmd -l 2-3 --proc-type=primary --file-prefix=pmd2 --vdev=net_memif -- -i
* lttng stop
* lttng destroy

---
## Analysis

### 🔥 **Flame Graph Analysis**


The flame graph reveals a structured and repetitive execution pattern in a network-processing application. The top-level function `pkt_burst_io_forward` dominates runtime, indicating it's the primary performance bottleneck. Functions like `common_fwd_stream_receive`, `rte_eth_rx_burst`, and `eth_memif_rx` appear consistently beneath it, reflecting their role in packet handling and forwarding. The repeated call stacks suggest steady, burst-based traffic processing. Deeper, short-lived functions likely handle utilities or parsing. Optimization should focus on `pkt_burst_io_forward` and its direct callees to achieve the most significant performance gains.

--- 

### 📊 **Counters Analysis (Thread-Level Performance Metrics)**


The **Counters** tab in Trace Compass provides real-time visualization of thread-level performance metrics over a selected time range. This view helps identify CPU usage patterns, instruction throughput, and cache behavior across threads.

The plot shows performance counters for four threads: `37211`, `37215`, `37228`, and `37232`. Each reports three key metrics:

-   `thread_cache_misses`
    
-   `thread_cpu_usage`
    
-   `thread_instructions`
    

The **Y-axis** shows the metric values, and the **X-axis** represents time, spanning from `16:38:51.614` to `16:38:51.682` (about 68 ms).


#### **Key Observations**

-   **Thread 37211** (black line) dominates the graph’s upper range, likely reflecting instruction count or CPU cycles. It remains near 2.4 million, suggesting high activity and tight-loop execution.
    
    -   **Brief dips** may indicate minor scheduling delays, cache stalls, or system interference.
        
-   Other threads (`37215`, `37228`, `37232`) show much lower counter values (~600k), indicating:
    
    -   They perform background or support tasks.
        
    -   They may be blocked or waiting for resources.
        
-   **Vertical lines** (likely marker events) appear periodically and align with activity spikes, potentially linked to packet bursts or scheduled tasks (consistent with the flame graph pattern).
    

#### **Interpretation and Implications**

-   The profile is **thread-skewed**, with thread 37211 consuming most resources.
    
-   Its high and stable counters suggest it may be CPU-bound or running uninterrupted real-time tasks.
    
-   Workload imbalance suggests potential for **load balancing** or **multi-threaded optimization**, by offloading work from 37211 to underused threads.
    

#### **Actionable Suggestions**

1.  **Investigate thread 37211**: Identify its role (e.g., main loop) and whether it's a bottleneck.
    
2.  **Check synchronization**: Ensure other threads aren’t blocked or starved.
    
3.  **Enable parallelism**: Distribute workload more evenly to improve CPU use.

---

### 🔢 **Statistics Overview (Counters)**

This view is showing aggregated counter data for the trace source `ust/uid/0/64-bit`.

#### 🔸 Columns:

-   **Level**: The trace source path (`ust/uid/0/64-bit`), representing the user-space tracing session for user ID 0 and architecture 64-bit.
    
-   **Events total**: Total number of recorded events in the entire trace: `22,785,417`.
    
-   **Events in selection**: Number of events in the currently selected time window or range: `1,215,466`.
    

----------

### 📊 **Pie Charts Analysis**

There are two pie charts:

1.  **Global**: Event distribution across all recorded data.
    
2.  **Events in selection**: Event distribution within the currently selected time interval.
    

#### 🟩 Green: `lttng_ust_cyg_profile:func_exit`

#### 🟨 Olive: `lttng_ust_cyg_profile:func_entry`

These two events are standard function tracing probes inserted by LTTng-UST using `cyg-profile`:

-   `func_entry`: Marks when a function is entered.
    
-   `func_exit`: Marks when a function returns.
    

The pie charts are almost 50/50 between `func_entry` and `func_exit`, indicating that for most function calls, there's a matching exit — a good sign of consistent tracing without dropped or unmatched events.

There are also minor slices for “Others,” which may include rare or user-defined events.

---

### 🌳 **Weighted Tree Viewer Analysis**

This view presents a _call tree_ where each function is nested inside the one that called it. It shows accumulated and self-time statistics over all invocations, helping you identify performance bottlenecks or hotspots.

####  Key Columns:

 **Function Tree (Leftmost)**  
    Functions are displayed in a hierarchical structure. For example:
    
photo (tree)
    
    This shows that `rte_memcpy` was called by `eth_memif_rx`, which was called by `rte_eth_rx_burst`, and so on — reflecting real call stack depth.
    
 **Duration**  
    Total cumulative time spent in a function including time spent in its children (nested calls).
    
    -   `pkt_burst_io_forward` → **1.275 s**
        
    -   `common_fwd_stream_receive` → **927.75 ms**
        
    -   `rte_eth_rx_burst` → **874.06 ms**
        
    
    This indicates that `pkt_burst_io_forward` is a high-level, long-running function.
    
  **Self time**  
    Time spent only in that function, excluding children.  
    This is useful to identify _computational hotspots_. For example:
    
    -   `rte_trace_point_fp_is_enabled` → **491.605 µs self time**, **491.605 µs total duration**
        
    -   `memif_get_ring_from_queue` → **26.649 ms self time**, meaning this function did actual computation without deeper calls.
        
 **Active CPU time**  
    Zero for all entries in this view, suggesting user-space profiling was focused on wall-clock time (likely due to lack of CPU-level sampling or specific tracepoint configuration).
    
 **Number of calls**  
    Total invocations across the trace. For instance:
    
    -   `rte_eth_rx_burst` → **259.8k**
        
    -   `rte_memcpy` → **154.1k**
        
    -   `rte_pktmbuf_reset` → **154.1k**
        

#### Insights:

-   The most expensive function by duration is `pkt_burst_io_forward`, which makes sense in a DPDK-style packet forwarding pipeline.
    
-   The function `rte_memcpy` appears frequently and with high cumulative duration, suggesting it's a potential optimization target.
    
-   Low-level tracing functions (like `rte_trace_feature_is_enabled`) appear many times but with negligible time.
    
-   Functions with high **self time** and high **call count** are likely worth reviewing for performance impact.

---

### 📊 Function Duration Distribution: Continuous Interpretation

####  **Left Panel (Table View)**

The table on the left lists individual function calls with the following columns:

-   **Start Time / End Time**: Indicates when each function started and ended. The timestamps show nanosecond precision.
    
-   **Duration**: Computed as the difference between end and start times. This is the function's total runtime.
    
-   **TID**: Thread ID of the executing thread, in this case consistently `372` (indicating focus on a specific thread, probably a worker or core thread in DPDK/VPP).
    

These raw values support granular inspection and are useful for tracing back specific spikes or anomalies seen in the histogram.


####  **Right Panel (Histogram View)**

This histogram shows:

-   **X-axis**: Function duration in microseconds (µs)
    
-   **Y-axis**: Number of times (count) a function ran for that duration
    

####  Interpretation of the Histogram:

  **Multi-modal Distribution**:  
    You see multiple **distinct peaks**, which indicates that there are several groups of function calls with different duration characteristics. This often reflects:
    
    -   Different code paths (e.g., fast path vs slow path)
        
    -   Varying packet sizes or processing complexity
        
    -   Conditional branching or waiting behavior in user code
        
 **High Peak at ~5–10 µs**:  
    The tallest bar (with count > 400,000) is concentrated in the lower microsecond range. These are ultra-fast function calls, likely from tight, efficient loops (possibly packet processing in a no-wait path).
    
**Secondary Peaks (~20–40 µs, ~60–80 µs, ~120 µs)**:  
    These peaks represent medium and higher latency operations, which could stem from:
    
    -   Buffer allocations
        
    -   Memory copy or checksum operations
        
    -   Occasional blocking calls or polling delays
        
 **Sparse Distribution Beyond 120 µs**:  
    Very few function calls exceed 120 µs, indicating that longer operations are rare — possibly related to exceptional cases like initialization or slow memory access.
    
**Logarithmic Y-axis**:  
    The Y-axis is logarithmic, making it easier to see the tail of rare longer-duration calls without the tall peak hiding them.
    


#### Key Takeaways

-   **Most function calls are very short (<20 µs)**, implying high performance and likely inline operations.
    
-   **Presence of multiple duration clusters** shows heterogeneous workloads or processing paths.
    
-   **Thread ID filtering** helps isolate one thread’s behavior for performance debugging or optimization.
    
-   **Useful for tuning**: Any peak at higher durations may indicate optimization candidates.

---
### 📊 Flame Chart

The **Flame Chart** offers a time-based view of function execution for thread **37232**, revealing a pattern of rapid, low-latency operations. Functions like `pkt_burst_io_forward`, `common_fwd_stream_receive`, and `rte_eth_rx_burst` dominate a critical interval (**16:38:51.620–16:38:51.680**), showing the packet processing loop in action. The frequent presence of `__rte_ethdev_trace_rx_burst_empty` indicates polling behavior. A specific trace shows a short-lived function (~32.8 µs), reflecting the system's fine-grained execution. Overall, the chart confirms a CPU-bound, high-throughput workload with tight loops and minimal idle time—characteristics typical of optimized DPDK environments.

-   Thread: **37232**
    
-   Time Window: `16:38:51.620` → `16:38:51.680`
    
-   Dominant functions:
    
    -   `pkt_burst_io_forward`
        
    -   `common_fwd_stream_receive`
        
    -   `rte_eth_rx_burst`
        

Nested calls & repetition show high-frequency, low-latency processing loop.

---

### 📋 Descriptive Statistics


The **Descriptive Statistics** view highlights which functions dominate execution time. Functions like `__rte_ethdev_trace_rx_burst_empty` and `pkt_burst_io_forward` are called frequently but are lightweight, while `common_fwd_stream_transmit` and `eth_memif_tx` have fewer calls yet high total durations. Memory management functions also contribute notable overhead. This mix of fast polling and heavier processing reveals key optimization targets in the application's packet processing path.


| Function                            | Calls | Avg Duration | Total Time |
|-------------------------------------|-------|--------------|------------|
| `__rte_ethdev_trace_rx_burst_empty` | 509k  | 513 ns       | 261 ms     |
| `common_fwd_stream_transmit`        | 11k   | 60.8 µs      | 666 ms     |
| `eth_memif_tx`                      | 10k   | 59.8 µs      | 655 ms     |
| `pkt_burst_io_forward`              | 520k  | 4.8 µs       | **2.5 s**  |

Insight: `pkt_burst_io_forward` dominates total runtime.


---

## 📈 Progress Summary
✅ Configured hugepages, built DPDK  
✅ Ran `helloworld` & `testpmd` with Memif  
✅ Integrated LTTng with `cyg-profile`  
✅ Performed full trace analysis in Trace Compass  
🔧 Remaining: Final test cases + optimization + documentation

---

## 🧩 Challenges & Solutions

**Problem:**  
VM-based Ubuntu showed only ~200 LTTng events → unusable trace data.

**Solution:**  
Switched to dual-boot native Ubuntu → full, accurate tracing with millions of events.
