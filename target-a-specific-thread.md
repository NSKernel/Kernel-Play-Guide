# Target a Specific Thread

Sometimes you just want to do some special actions for the thread you created, or just want to add a value to a thread. Pretty obvious you are going to add a field to the thread's metadata. In this section we will discuss just how to do that.

## task\_struct - A Thread's Metadata

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

## Initializing a task\_struct

Now you have your fields, you need to be responsible for them. You should have known that except for the very first thread, every thread is born from `fork`. So now things are clear. Two _tasks_: Initialize the first thread, deal with `fork`.

### Initialize the very first thread



