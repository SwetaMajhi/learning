Hardware and Operating Systems

The origin of cache!!
- the exponentially increasing number of transistors was initially used for faster and faster clock speed. Faster clock speed means more instructions completed per second.
- Issue: Faster chips require a faster stream of data to act upon, over time main memory could not keep up with the demands of the processor core for fresh data.
- Problem: if the CPU is waiting for data, then faster cycles don’t help, as the CPU will just have to idle until the required data arrives.
- Solution: CPU caches were introduced.

Memory cache
- Memory areas on the CPU that are slower than CPU registers, but faster than main memory.
- CPU fills the cache with copies of often accessed memory locations, rather than asking main memory
- several layers of cache, most-often-accessed caches being located close to the processing core.
- L1=cache closest to the CPU. Typically, each core has dedicated L1 and L2 caches, while L3 is shared across cores. Lower levels are faster but smaller.
- Caches significantly improve data access times and help keep the processor supplied with data, reducing idle waiting time. A larger portion of transistors is now allocated to building bigger, more sophisticated caches rather than just boosting clock speed further
- Main memory is accessed over the Northbridge component, and it is traversing this bus that causes the large drop-off in access time to main memory. (img)
![Overall CPU and memory architecture](image.png)

- Problem: Multiple caches create a problem: ensuring memory consistency when data is cached in different places.
- Solution: cache consistency protocols.

MESI Protocol 
- A common solution defines four cache states for any line in a cache. Each line (usually 64 bytes) is either:
	- Modified: Changed, not yet written to main memory
	- Exclusive: Only in this cache, matches main memory
	- Shared: May exist in other caches, matches main memory
	- Invalid: Cannot be used; will be discarded
** General working idea of the protocol is {Later}
	//something like:
	MESI tracks which cache (at any level) has which version of data
	When Core 1 modifies data in its L1/L2, it broadcasts on the memory bus
	Core 2's L1/L2 listen and invalidate their copies
	Core 2 then fetches the latest data from: Core 1's cache (via L3) or main memory
- Cache Write Strategy: 
	- Early approach where every cache write goes directly to main memory (inefficient, high bandwidth) = Write-through
	- Modern approach where only modified cache blocks are written to memory when replaced (significantly reduces traffic) = Write-back = performance optimization that trades safety for speed.

Translation Lookaside Buffer (TLB):
- Acts as a cache for for the page tables = virtual-to-physical memory address mappings
- Without TLB, virtual address lookups would take 16 cycles, (unacceptable performance) even if the page table was held in the L1 cache.

Branch Prediction & Speculative Execution
- Problem: Conditional branches stall the pipeline (up to 20 cycles) because the CPU doesn't know which instruction comes next until the condition is evaluated
- Solution: CPU uses transistors to build a heuristic that predicts which branch is likely taken
- How it works: CPU speculatively fills the pipeline based on the prediction; if correct, it continues normally; if wrong, partially-executed instructions are discarded and the pipeline is flushed
- Tradeoff: Gamble on performance vs. penalty for misprediction
//is it like we should avoid conditional statements while programming? NO.  branch prediction works very well when branches are predictable.

Hardware Memory Models
- The compiler (javac), JIT, and CPU are all allowed to reorder instructions for performance optimization — as long as the reordering doesn't affect what the current thread observes. 
- Ex.	myInt = otherInt;   // Step 1
		intChanged = true;  // Step 2
From the current thread's perspective, there's nothing between these two lines, so it doesn't matter which runs first. But from another thread's perspective, this is dangerous, it might see intChanged == true (Step 2 ran), yet still read the old value of myInt (Step 1 hasn't run yet).
- Java's solution:
	- Java deliberately uses a weak memory model to stay portable across all processor types. It doesn't assume strong consistency from hardware.
	- Instead, it puts the responsibility on the developer to use:
		- synchronized / locks — to establish happens-before relationships
		- volatile — to ensure visibility of changes across threads
**The term mechanical sympathy has been coined by Martin Thompson and others to describe this approach, especially as applied to the low-latency and high-performance spaces. It can be seen in recent research into lock-free algorithms and data structures, which we will meet toward the end of the book.

Operating System
- Core Purpose: An OS arbitrates access to shared, finite resources — primarily memory and CPU time — across multiple competing processes.
- Memory Management: 
	- The MMU + page tables provide virtual addressing, isolating each process's 
	- The MMU operates too low-level for developers to directly influence
	- TLBs = is hardware feature = speeds up physical memory lookups
- Process Scheduler
	- Controls who gets CPU time and when, using a run queue — a waiting list of threads ready to run but waiting their turn.
	- Java threads map directly to OS threads in practice (older "green threads" approaches were abandoned)
	- Modern machines have multiple cores, enabling true parallel execution — making reasoning about concurrency complex.
	- Ex.
	long start = System.currentTimeMillis(); for (int i = 0; i < 2_000; i++) { Thread.sleep(2);} 
	long end = System.currentTimeMillis(); System.out.println("Millis elapsed: " + (end - start) /4000.0);
	"...Earlier versions of Windows had notoriously bad schedulers, with some versions of Windows XP reporting up to 180% overhead for scheduling (so that 1,000 sleeps of 1 ms would take 2.8 s). ..."
	- When you call Thread.sleep(1ms), the thread goes to the back of the run queue. The actual time it sleeps depends on:
		- How long until it reaches the front of the queue again
		- How many other threads are competing
		- The scheduler's time quantum (10–100ms)
	So sleep(1ms) could actually take 2–3ms in reality.
- Timing Across Operating Systems
	- System.currentTimeMillis() in Java ultimately calls os::javaTimeMillis() — a native, OS-specific function. Implementations vary significantly like BSD/macOS, Linux, Solaris, AIX call "gettimeofday()", Windows calls "GetSystemTimeAsFileTime()". Windows also has a fake timer mode (UseFakeTimers) and variable clock accuracy depending on available hardware — making Java timing calls highly unpredictable on Windows.
- Context Switching
	- is OS scheduler swapping out a running thread and replacing it with a waiting one. This involves saving/restoring:
		- Executing instructions
		- Stack state
	- "....A context switch can be a costly operation, whether between user threads or from user mode into kernel mode (sometimes called a mode switch). The latter case is particularly important, because a user thread may need to swap into kernel mode in order to perform some function partway through its time slice...."
	- Impact of mode switching: User space and kernel space access completely different memory regions. So when the CPU changes its execution mode to kernel mode:
		- Instruction cache = Flushed -> Kernel runs completely different code
		- Data cache = Flushed -> Kernel accesses different memory
		- TLBs = Invalidated  -> Virtual address mappings are different
		- caches now COLD (slow!) must refill with user data again
		-  True Cost "Masked": Because when you measure a system call like: 
			javalong start = System.nanoTime();
			// system call happens here
			long end = System.nanoTime();
		You only measure the kernel execution time. You don't capture the post-return slowdown while caches refill.
	- To mitigate impact of mode switching --> Linux's solution --> vDSO (virtual dynamically shared object) = a Linux mechanism that maps certain kernel data structures directly into user space memory, avoiding a full kernel mode switch for safe, side-effect-free syscalls.
		- Normally: user → kernel mode switch → read clock → return → cache refill penalty
		  With vDSO: just reads mapped memory directly in user space — no switch, no penalty
		- Timing and context switching are deeply OS-dependent. What looks like a simple System.currentTimeMillis() call can carry wildly different costs across platforms — and on any platform, frequent kernel switches silently degrade performance through cache invalidation.

- Any of these aspects of a system can be responsible for a performance problem:
	- Hardware & OS the application runs on
	- JVM (or container) the application runs in
	- The application code itself
	- External systems the application calls
	- Incoming request traffic hitting the application

- Basic detection strategies
	- A well-performing application makes efficient use of system resources: CPU, memory, network, and I/O bandwidth. Hitting any resource limit causes a performance problem. Identify which resource limit is being hit — then either increase the available resources or improve efficiency of use.
	- The OS manages resources on behalf of user processes — it should not be a major consumer itself. Exception: when I/O or memory requirements greatly exceed hardware capability.
	- CPU Utilization
		- Applications should be aiming for as close to 100% usage as possible during periods of high load. CPU cycles are quite often the most critical resource needed by an application, and so efficient use of them is essential for good performance.
		- vmstat and iostat = command-line tools to provide immediate insight into the current state of the virtual memory and I/O subsystems, respectively --> provide numbers at the level of the entire host, but good enough to go point whether need to dive deeper 
		- If CPU is not near 100% user time, for CPU bound activities(Matrix multiplication, Cryptography / hashing, Sorting large in-memory datasets, Video encoding/compression, etc) + high context switch rate = likely due to :
			- Blocking due to I/O contention
			- Thread lock contention --> tools like VisualVM, statistical thread profiler can help
	- Garbage collection
		- HotSpot JVM allocates memory at startup and manages it entirely from user space.
		- No system calls like sbrk() are needed, so GC causes minimal kernel-switching activity. 
		- GC burns user space CPU cycles, not kernel space cycles --> If 100% CPU usage, mostly user space, but throughput is low / latency is high, GC is likely thrashing, burning cycles on collection instead of useful work --> Check GC logs for how frequently entries are being added.
		- GC logging is extremely cheap and highly valuable for performance analytics.
		- GC logging should be enabled on all JVM processes, especially in production.
	- I/O Monitoring
		- Unlike memory, which has the clean abstraction of virtual memory, I/O has no comparable isolation mechanism for app developers.
		- Production engineers already tend to actively monitor heavy I/O usage as standard practice.
		- Tools like iostat and vmstat provide basic counters (blocks in/out) that are usually enough for basic diagnosis, assuming one I/O-intensive app per host.
		*** Kernel bypass IO:{Later}
	- Mechanical sympathy
		- The idea that understanding the underlying hardware leads to better performance.
		- For many Java developers, the JVM abstracts away hardware concerns — but this abstraction also makes reasoning about performance harder. Developers can still succeed in high-performance/low-latency Java by understanding how the JVM interacts with hardware.
		- The case of cacheline:
			- Processors use cache lines to fetch blocks of memory, not individual bytes.
			- In a multithreaded context: if two threads read/write variables on the same cache line, they compete for it — even if they're using different variables.
			- Thread A modifies the cache line → invalidates it for Thread B → Thread B re-reads from memory → Thread A's line is invalidated again, and so on. This ping-pong is false sharing.
			- Fix: Add padding around variables to force them onto separate cache lines. 

- Virtualization
	- Virtualization runs a copy of an OS as a single process on top of an already-running OS (the "host"), which itself runs on bare metal.
	- On a normal OS, the kernel runs in privileged mode with direct hardware access. On a virtualized system, guest OS direct hardware access is prohibited. Privileged instructions are rewritten as unprivileged instructions.
	- The JVM provides a portable execution environment = abstracting away the OS for Java code. But for fundamental services (thread scheduling, system clock, etc.), the underlying OS must be accessed directly = done via native methods = marked with the native keyword, written in C, but callable as ordinary Java methods. This interface = the Java Native Interface (JNI).
 	- Example: System.currentTimeMillis()
	- The Java method System.currentTimeMillis() is implemented in C++ as os::javaTimeMillis().
	- It is exposed to Java via a C bridge using JNI.
	- The mapping chain: System.currentTimeMillis() → JVM_CurrentTimeMillis() (C function with C++ internals) → os::javaTimeMillis().
	- os::javaTimeMillis() is OS-dependent; its implementation lives in OS-specific subdirectories within the OpenJDK source.
	This is how Java's platform-independent layer calls into OS/hardware-specific services when needed.


