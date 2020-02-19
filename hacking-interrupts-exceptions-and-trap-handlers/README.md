# Hacking Interrupts, Exceptions and Trap Handlers

On x86-64, several things could take a user program to another world - the kernel space. You have interrupts, exceptions and traps. All of these works basically the same except for some tiny differences. 

When any of these happens, the CPU will pause the current task, save the register states to the IRQ stack, find the corresponding handler in the kernel and let that handler handle the upcomming things. You can find more on the web.

In this chapter, we will be mainly discussing the shitty and tricky things you might meet when hacking a handler.



 

