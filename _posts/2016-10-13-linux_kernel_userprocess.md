---
layout: post
title:  "内核空间运行用户程序"
date:   2016-10-13 17:09:35
categories: [编程, Linux, Kernel]
tags: [Linux, Kernel, ]
description: ""
---

系统初始化时kernel_init在内核态创建和运行应用程序以完成系统初始化.  内核刚刚启动时，只有内核态的代码，后来在init过程中，在内核态运行了一些初始化系统的程序，才产生了工作在用户空间的进程。

    /* This is a non __init function. Force it to be noinline otherwise gcc
    736 * makes it inline to init() and it becomes part of init.text section
    737 */
    738 static noinline int init_post(void)
    739{
    740        /* need to finish all async __init code before freeing the memory */
    741        async_synchronize_full();
    742        free_initmem();
    743        mark_rodata_ro();
    744        system_state = SYSTEM_RUNNING;
    745        numa_default_policy();
    746
    747
    748        current->signal->flags |= SIGNAL_UNKILLABLE;
    749
    750        if (ramdisk_execute_command) {
    751                run_init_process(ramdisk_execute_command);
    752                printk(KERN_WARNING "Failed to execute %s\n",
    753                                ramdisk_execute_command);
    754        }
    755
    756        /*
    757         * We try each of these until one succeeds.
    758         *
    759         * The Bourne shell can be used instead of init if we are
    760         * trying to recover a really broken machine.


从内核里发起系统调用，执行用户空间的应用程序。这些程序自动以root权限运行。


    761         */
    762        if (execute_command) {
    763                run_init_process(execute_command);
    764                printk(KERN_WARNING "Failed to execute %s.  Attempting "
    765                                        "defaults...\n", execute_command);
    766        }
    767        run_init_process("/sbin/init");
    768        run_init_process("/etc/init");
    769        run_init_process("/bin/init");
    770        run_init_process("/bin/sh");
    771
    772        panic("No init found.  Try passing init= option to kernel. "
    773              "See Linux Documentation/init.txt for guidance.");
    774}

这里，内核以此运行用户空间程序，从而产生了第一个以及后续的用户空间程序。一般用户空间的init程序，会启动一个shell，供用户登录系统用。这样，这里启动的用户空间的程序永远不会返回。也就是说，正常情况下不会到panic这一步。系统执行到这里后，Linux Kernel的初始化就完成了。
此时，中断和中断驱动的进程调度机制，调度着各个线程在各个CPU上的运行。中断处理程序不时被触发。操作系统上，一些内核线程在内核态运行，它们永远不会进入用户态。它们也根本没有用户态的内存空间。它的线性地址空间就是共享内核的线性地址空间。一些用户进程通常在用户态运行。有时因为系统调用而进入内核态，调用内核提供的系统调用处理函数。

但有时，我们的内核模块或者内核线程希望能够调用用户空间的进程，就像系统启动之初init_post函数做的那样。

如，一个驱动从内核得到了主从设备号，然后需要使用mknod命令创建相应的设备文件，以供用户调用该设备。

如，一个内核线程想神不知鬼不觉地偷偷运行个有特权的后门程序。等等之类的需求。

**call_usermodehelper函数**
    Linux  Kernel提供了call_usermodehelper函数，让我们能够异常方便地在内核中直接新建和运行用户空间程序，并且该程序具有root权限。

**call_usermodehelper函数源码**
include/linux/kmod.h头文件

    105static inline int
    106call_usermodehelper(char *path, char **argv, char **envp, enum umh_wait wait)
    107{
    108        return call_usermodehelper_fns(path, argv, envp, wait,
    109                                       NULL, NULL, NULL);
    110}
    111
   
   
     50enum umh_wait {
     51        UMH_NO_WAIT = -1,       /* don't wait at all */
     52        UMH_WAIT_EXEC = 0,      /* wait for the exec, but not the process */
     53        UMH_WAIT_PROC = 1,      /* wait for the process to complete */
     54};
     55
     56struct subprocess_info {
     57        struct work_struct work;
     58        struct completion *complete;
     59        char *path;
     60        char **argv;
     61        char **envp;
     62        enum umh_wait wait;
     63        int retval;
     64        int (*init)(struct subprocess_info *info);
     65        void (*cleanup)(struct subprocess_info *info);
     66        void *data;
     67};
     68

kernel/kmod.c实现文件

    377/**
    378 * call_usermodehelper_exec - start a usermode application
    379 * @sub_info: information about the subprocessa  子进程的信息
    380 * @wait: wait for the application to finish and return status.等待用户空间子进程的完成，并返回结果。
    381 *        when -1 don't wait at all, but you get no useful error back when
    382 *        the program couldn't be exec'ed. This makes it safe to call
    383 *        from interrupt context.


-1表示根本不等待子进程的结束。 但这样你就无法对程序出错进行处理。
如果使用中断上下文，那么应该使用-1。

    384 *
    385 * Runs a user-space application.  The application is started
    386 * asynchronously if wait is not set, and runs as a child of keventd.
    387 * (ie. it runs with full root capabilities).

call\_usermodehelper_exec函数，启动一个用户模式应用程序。
如果不设置wait，那么用户空间应用程序会被异步启动。  它在root权限下运行。是keventd进程的子进程。

    388 */
     389int call_usermodehelper_exec(struct subprocess_info *sub_info,
     390                             enum umh_wait wait)
     391{
     392        DECLARE_COMPLETION_ONSTACK(done);
     393        int retval = 0;
     394
     395        helper_lock();
     396        if (sub_info->path[0] == '\0')
     397                goto out;
     398
     399        if (!khelper_wq || usermodehelper_disabled) {
     400                retval = -EBUSY;
     401                goto out;
     402        }
     403
     404        sub_info->complete = &done;
     405        sub_info->wait = wait;
     406把用户空间进程挂到一个内核工作队列。
     407        queue_work(khelper_wq, &sub_info->work);
     408        if (wait == UMH_NO_WAIT)        /* task has freed sub_info */
     409                goto unlock;

如果等待子进程完成，那么执行等待完成的  事件通知和唤醒。就是说当前进程sleep。
 
     410        wait_for_completion(&done);
     411        retval = sub_info->retval;
     412
     413out:
     414        call_usermodehelper_freeinfo(sub_info);
     415unlock:
     416        helper_unlock();
     417        return retval;
     418}
     419EXPORT_SYMBOL(call_usermodehelper_exec);
     420
     421void __init usermodehelper_init(void)
     422{
     423        khelper_wq = create_singlethread_workqueue("khelper");
     424        BUG_ON(!khelper_wq);
     425}
 
call_usermodeheler函数创建的新程序，实际上作为keventd内核线程的子进程运行，因此具有root权限。  新程序被扔到内核工作队列“khelper”中进行执行。

如果使用UMH_NO_WAIT，那么因为没有在事件队列上等待和唤醒的过程，因此可以在中断上下文中使用。 它的返回值是新程序的返回值。

call_usermodeheler函数的参数用法和execve函数一致 

    #include<unistd.h>

    intexecve(const char *filename, char *const argv[],
    char*const envp[]);

execve函数使用sys_execve系统调用，创建并运行一个程序。

argv是字符串数组，是将被传输到新程序的参数。

envp是字符串数组，格式是key=value，是传递给新程序的环境变量。

argv和envp都必须以NULL字符串结束。以此来实现对字符串数组的大小统计。 

这就意味着，argv的第一个参数也必须是程序名。也就是说，新程序名要在execve函数的参数中传递两次。

这和main函数传入的参数格式也是一致的。

    #include <linux/init.h>
    #include <linux/module.h>
    #include <linux/moduleparam.h>
    //#include<linux/config.h>
    
    #include <linux/kernel.h>/*printk()*/
    #include <linux/sched.h>
    
    MODULE_LICENSE("GPL");
    
    
    static __init int test_driver_init(void)
    {
        int result = 0;
        char cmd_path[] = "/usr/bin/touch";
        char* cmd_argv[] = {cmd_path,"/touchX.txt",NULL};
        char* cmd_envp[] = {"HOME=/", "PATH=/sbin:/bin:/usr/bin", NULL};
    
        result = call_usermodehelper(cmd_path, cmd_argv, cmd_envp, UMH_WAIT_PROC);
        printk(KERN_DEBUG "test driver init exec! there result of call_usermodehelper is %d\n", result);
        printk(KERN_DEBUG "test driver init exec! the process is \"%s\", pid is %d.\n",current->comm, current->pid);
        return result;
    }
    
    
    static __exit void test_driver_exit(void)
    {
    int result = 0;
    char cmd_path[] = "/bin/rm";
    char* cmd_argv[] = {cmd_path,"/touchX.txt",NULL};
    char* cmd_envp[] = {"HOME=/", "PATH=/sbin:/bin:/usr/bin", NULL};

    result = call_usermodehelper(cmd_path, cmd_argv, cmd_envp, UMH_WAIT_PROC);
    printk(KERN_DEBUG "test driver exit exec! the result of call_usermodehelper is %d\n", result);
    printk(KERN_DEBUG "test driver exit exec! the process is \"%s\",pidis %d \n", current->comm, current->pid);
    }

    module_init(test_driver_init);
    module_exit(test_driver_exit);


{% highlight ruby %}
文档信息
--------------
* 版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
* [转载: ](blog.163.com/cupidove/blog/static/1005662)
{% endhighlight %}

