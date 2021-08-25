# Debugging Memory Leaks in Urbit

The most common issue with memory is the infamous segfault (`SIGSEGV`).  There are also a variety of memory leaks which need to be carefully handled in a reference-counted system like the Urbit jets.

Your tools include:

1.  "Pen and paper" accounting.  Triple-check reference counts.  In particular, make sure to `u3a_free` anything you `u3a_malloc`, and `u3z` anything `u3_noun`s or `u3_atom`s you are done with and won't be `return`ing.  Refcounts are otherwise more of an issue in system or legacy jets which use transfer semantics (`u3k`).  You are likely working with retain semantics (`u3q`/`u3w`) which seem to be more intuitive for those of us not versed in Objective-C.

    Similarly, check all memory writes carefully.  If you `u3a_malloc` an array, but inadvertently overwrite the bounds, then when you `u3a_free` it or pass it you will see loom corruption.  While you are in the function, however, it is likely to behave as you expect, making it difficult to trace down.
    
    You can unwrap loom pointers into structs using `u3a_to_ptr`, which gives you direct access to `len_w` and `buf_w`.  Use this wisely.

2.  Valgrind and ASan.  Memory debugging frameworks run the entire program in an instrumented environment that provides details about allocations and frees.  The real problem is that Urbit's `mmap` is too large for any real system to have the Valgrind multiplier on top of it.  You'll need to reduce memory usage in `pkg/urbit/vere/disk.c` from `1099511627776` to `64424509440`.  Then enable memory debugging in `nix/pkgs/urbit/default.nix`.  For me, this looked like:

    ```sh
    #CFLAGS = [ (if enableDebug then "-O0" else "-O3") "-g" ]
    CFLAGS = [ "-O0" "-g" ]
      #++ [ "-Werror" ]  # -O0 causes warnings
      ++ lib.optionals enableStatic [ "-static" ];

    MEMORY_DEBUG = true;
    CPU_DEBUG = enableDebug;
    EVENT_TIME_DEBUG = false;
    ```

    When you run Valgrind, use these options:

    ```sh
    valgrind --leak-check=yes --show-leak-kinds=all --track-origins=yes urbit -g zod
    ```

    As Valgrind runs much more slowly than plain-vanilla Urbit, this may take some time to complete.
    
    (I have not used ASan as it requires compile-time integration.)

3.  `gdb`.  The debugger is able to catch termination signals or at least provide a framework for stack traces after-the-crash.

    The Urbit runtime is divided into the king and the serf.  The king is responsible for implementing the Arvo event log and event loop.  The serf evaluates Nock and handles jets.  Since Urbit runs with a king process and a serf process, you must attached to these individually or you will miss some subset of events.

    1. Start the king process (optionally with `gdb`, but for jets this typically isn't necessary).  If in `gdb`, set up your breakpoints and run the ship:  `r zod`
    2. Attach to the `urbit-worker` serf process with `gdb`.  (This requires `sudo` on my Linux system.)  Use `ps -elf | grep urbit-worker` to locate the PID.
    3. Set breakpoints and handlers on the worker process `gdb`.  Then `c` to continue.  For instance, to handle a segfault, use `handle SIGSEGV nostop noprint` to invoke `u3e_fault` normally but catch the error and get the stack trace.  (If you need addresses as well as function names, use `objdump -d urbit` to obtain these.)
    4. If using `gdb` on the king, run the main process.
    5. Invoke the generator, test, or code which causes the memory problem and observe what happens in the serf `gdb` process.

    If you are setting breakpoints and they are not being hit by `gdb`, most likely you don't have the king or serf `gdb` process set up.

    If you need to break out of the process back into `gdb`, use `Ctrl`+`C`.

    If you want to set a manual breakpoint in the code itself, use `SIGABRT`:

    ```c
    #include <signal.h>

    raise(SIGABRT);
    ```

    with a handler in `gdb`.

    `lldb` is [reputedly](https://stackoverflow.com/questions/26529581/lldb-break-upon-sigsegv) more fragile when dealing with memory issues such as segfaults and isn't preferred.

4.  Documentation.  Triple-check your assumptions about allocate, imprison, retrieve, and xtract functions.  For instance, I've had my share of problems with `u3i_words` which expects a _length_ not an _end index_; for some reason this is counterintuitive to me.

You can exploit the separation between the runtime and the event log to save some time booting ships and jamming pills.  You can create a ship from a pill and continue to reuse it until it becomes unstable for development-driven reasons.  This applies even through repeated modifications and builds of the runtime.  I normally make a new `zod`, set up `|mount %` and copy in necessary files, then copy it to a folder `zod-ist-tot` as a backup.
