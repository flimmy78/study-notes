---
tags : [Linux , Kernel]
category : [ 读核 ]
---

kernel 初始化过程
===

内核启动后从汇编进入 `start_kernel()` ，然后会调用 `rest_init()` ，该函数会创建一个内核进程运行 `kernel_init()` :

```
static noinline void __init_refok rest_init(void)
{
...
    /*
     * We need to spawn init first so that it obtains pid 1, however
     * the init task will end up wanting to create kthreads, which, if
     * we schedule it before we create kthreadd, will OOPS.
     */
    kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
...
}
```

`kernel_thread()` 的原型是 ` pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags);` ，创建一个内核线程。

`kernel_init()` 会创建 `init` 内核线程执行 init 任务：


```
static int __ref kernel_init(void *unused)
{
    ...
    if (ramdisk_execute_command) {
        if (!run_init_process(ramdisk_execute_command))
            return 0;
        pr_err("Failed to execute %s\n", ramdisk_execute_command);
    }
    if (execute_command) {
        if (!run_init_process(execute_command))
            return 0;
        pr_err("Failed to execute %s.  Attempting defaults...\n", execute_command);
    }
    if (!run_init_process("/sbin/init") ||
        !run_init_process("/etc/init") ||
        !run_init_process("/bin/init") ||
        !run_init_process("/bin/sh"))
        return 0;

    panic("No init found.  Try passing init= option to kernel. "
          "See Linux Documentation/init.txt for guidance.");
}
```

内核会根据实际情况、顺序选择执行 `ramdisk_execute_command` 、 `execute_command` 、 `/sbin/init` 、 `/etc/init` 、 `/bin/init` 、 `/bin/sh`，只要一个执行成功就会返回成功，否则进入 `panic()`。

（未完待续）
