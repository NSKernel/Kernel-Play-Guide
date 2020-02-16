# Target a Specific Thread

Sometimes you just want to do some special actions for the thread you created, or just want to add a value to a thread. Pretty obvious you are going to add a field to the thread's metadata. In this section we will discuss just how to do that.

## `task_struct` - A Thread's Metadata

In Linux Kernel, a thread is also known as a task. Process is a view from resource management, while threads, well, is a vew from thread management. Linux stores each thread's metadata inside a struct called `task_struct`. The definition of `task_struct` can be found in `/include/linux/sched.h`. As you can see, the whole struct contains tons of fields that are inside some `#ifdef CONFIG_XXX_XXX`, so this is also a warning to you that you should not define yours in them unless you know what you are doing.

## Your Own Field in a Thread

It's not my job to teach you how to add a field to a struct in C language. This section only contains tips.

* After a few lines from the start of `task_struct`, you will see something like this. Follow the comment and add your field after this.

  ```c
    /*
  	 * This begins the randomizable portion of task_struct. Only
  	 * scheduling-critical items should be added above here.
  	 */
  	randomized_struct_fields_start
  ```

* If you are just adding a generic field into the `task_struct`, be careful not to add your field into areas of a `#ifdef CONFIG_XXX_XXX`.
* Be sure that you are not making the `task_struct` too big.

The third tip is obvious. The first two tips could boil down to a simple and brainless rule: always add your fields directly after `randomized_struct_fields_start`.

## Initializing a `task_struct`

Now you have your fields, you need to be responsible for them. You should have known that except for the very first thread, every thread is born from `fork`. So now things are clear. Two _tasks_: Initialize the first thread, deal with `fork`.

### Initialize the very first thread

The very first task is called the `init_task`. You may find it in `/init/init_task.c`. It's pretty much a classic C struct variable initialization. Nothing too much to say, add your fields and it's done.

Note that in early versions of the kernel, `init_task` is located at `/init/init_task.c`, and is defined as a macro `#define INIT_TASK(tsk)`. 

### Deal with fork

`fork` will clone one thread into two, so the `task_struct` is also copied. The behaviour of copying the `task_struct` happens in `copy_process` in `/kernel/fork.c`. `copy_process` is a really huge function that does tons of dirty works for us. Understanding it is not easy but we can quickly obeserve that several tens of lines later we find a lot of `p->xxx = xxxx`, which _seems_ to be the place where the `task_struct` copying process happens. And indeed it is. Now you just add `p->your_field = your_value` and it's done. Do remember to be careful with `#ifdef CONFIG_XXX_XXX`.

If you want to inherit field value from the parent task\_struct, you can do

```text
    p->your_field = current->your_field;
```

For the meaning of `current`, read the following section.

## `current` - Current `task_struct`

In a lot of time you want to access the `task_struct` of the current thread. Linux provides you with `current` identifier. To use it, include `linux/sched.h`. `current` is per-CPU defined so every thread on its own CPU will get its own `task_struct`.

