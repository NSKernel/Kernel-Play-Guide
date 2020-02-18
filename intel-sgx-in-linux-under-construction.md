# Intel SGX in Linux - Under Construction

Intel SGX provides something that is known as an _enclave_. Basically a secret room in which no one else knows what's happening inside. The memory of it is fully encrypted and integrity-checked, and every time it exits the CPU will flush all buffer and cache. So if you are doing something like an SSH key computing then SGX will be a very safe place to do it.

The code inside the enclave works like a shared library except that they are called not by the `call` instruction but rather something else. Of course, the code inside the enclave can access data in the outside and can also call outside libraries to achieve certain functions.

## ECalls and OCalls

Invoking an oridary function is called a _call_ to a function. So similarly, invoking an enclave function is now called as an ECall. Inside an enclave, invoking an outside function is now called as an OCall.

## EENTER - Enter an Enclave

In SGX, you call an enclave function using `eenter`. This instruction \(?, more on that later\) will take you to the enclave function you want. Basically you provide 2 parameters, one is called a TCS \(Thread Control Structure\), another is called an AEP \(Asynchronous Exit Pointer\).

