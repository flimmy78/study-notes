
initcall
=========

<!-- MarkdownTOC -->

- 1. initcall 定义
- 2. initcall 的执行

<!-- /MarkdownTOC -->


## 1. initcall 定义

```
#define early_initcall(fn)      __define_initcall(fn, early)
#define pure_initcall(fn)       __define_initcall(fn, 0)

#define core_initcall(fn)       __define_initcall(fn, 1)
#define core_initcall_sync(fn)      __define_initcall(fn, 1s)
#define postcore_initcall(fn)       __define_initcall(fn, 2)
#define postcore_initcall_sync(fn)  __define_initcall(fn, 2s)
#define arch_initcall(fn)       __define_initcall(fn, 3)
#define arch_initcall_sync(fn)      __define_initcall(fn, 3s)
#define subsys_initcall(fn)     __define_initcall(fn, 4)
#define subsys_initcall_sync(fn)    __define_initcall(fn, 4s)
#define fs_initcall(fn)         __define_initcall(fn, 5)
#define fs_initcall_sync(fn)        __define_initcall(fn, 5s)
#define rootfs_initcall(fn)     __define_initcall(fn, rootfs)
#define device_initcall(fn)     __define_initcall(fn, 6)
#define device_initcall_sync(fn)    __define_initcall(fn, 6s)
#define late_initcall(fn)       __define_initcall(fn, 7)
#define late_initcall_sync(fn)      __define_initcall(fn, 7s)
```

最终都是通过 `__define_initcall` 实现的 ：

```
#define __define_initcall(fn, id) \
    static initcall_t __initcall_##fn##id __used \
    __attribute__((__section__(".initcall" #id ".init"))) = fn
```

也就是说经过 `*_initcall` 修饰的函数最终都会按照顺序放到 section `.initcall#id#.init` 中，然后kernel 会按照顺序执行这些函数。
 
而在 `include/asm-generic/vmlinux.lds.h` 里面已经定义好了对应的 section ：

```
#define INIT_DATA_SECTION(initsetup_align)              \
    .init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {       \
        INIT_DATA                       \
        INIT_SETUP(initsetup_align)             \
        INIT_CALLS                      \
        CON_INITCALL                        \
        SECURITY_INITCALL                   \
        INIT_RAM_FS                     \
    }
#define VMLINUX_SYMBOL(x) __VMLINUX_SYMBOL(x)

#define INIT_CALLS_LEVEL(level)                     \
        VMLINUX_SYMBOL(__initcall##level##_start) = .;      \
        *(.initcall##level##.init)              \
        *(.initcall##level##s.init)             \

#define INIT_CALLS                          \
        VMLINUX_SYMBOL(__initcall_start) = .;           \
        *(.initcallearly.init)                  \
        INIT_CALLS_LEVEL(0)                 \
        INIT_CALLS_LEVEL(1)                 \
        INIT_CALLS_LEVEL(2)                 \
        INIT_CALLS_LEVEL(3)                 \
        INIT_CALLS_LEVEL(4)                 \
        INIT_CALLS_LEVEL(5)                 \
        INIT_CALLS_LEVEL(rootfs)                \
        INIT_CALLS_LEVEL(6)                 \
        INIT_CALLS_LEVEL(7)                 \
        VMLINUX_SYMBOL(__initcall_end) = .;

```

最终编译内核使用 lds 链接脚本 `$(arch)/kernel/vmlinux.lds` 中就有：

```
INIT_DATA_SECTION(...)
```

## 2. initcall 的执行

kernel 启动过程中会执行 `init/main.c` 的 `start_kernel()`，然后按照下面的调用链最终会调用 `do_initcalls()` 执行这些初始化函数：

`start_kernel()`->`rest_init()`->`kernel_thread(kernel_init)`->`kernel_init_freeable()`->`do_basic_setup()`->`do_initcalls()`->`do_initcall_level()`

而执行的顺序就跟各个宏的定义有关，比如 `pure_initcall` 实际上是 `__define_initcall(fn, 0)` 所以它会被首先执行，而 `late_initcall` 是 `__define_initcall(fn, 7)` 所以会最后执行。具体的执行如下：

```
static void __init do_initcalls(void)
{
    int level;

    for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
        do_initcall_level(level);
}
```

`do_initcall_level()` 会执行每个 section 内部的所有初始化函数 ：

```
static void __init do_initcall_level(int level)
{
...
    for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
        do_one_initcall(*fn);
}
```

`do_initcall_level()` 按顺序遍历每个 section 执行所有函数。


两个相关的数组，`initcall_levels` 指向了每个 section 的起始地址，也就是第一个函数地址，`initcall_level_names` 则将 section 的顺序号（就是 0,1,2,3 这些）和初始化函数的含义关联起来（比如 0 对应 early ，1 对应 core）：

```
static initcall_t *initcall_levels[] __initdata = {
    __initcall0_start,
    __initcall1_start,
    __initcall2_start,
    __initcall3_start,
    __initcall4_start,
    __initcall5_start,
    __initcall6_start,
    __initcall7_start,
    __initcall_end,
};

/* Keep these in sync with initcalls in include/linux/init.h */
static char *initcall_level_names[] __initdata = {
    "early",
    "core",
    "postcore",
    "arch",
    "subsys",
    "fs",
    "device",
    "late",
};
```
















