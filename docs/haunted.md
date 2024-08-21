# How Meltdown and Spectre haunt Anti-Cheat

## Abstract
Puzzled by our observation of modified versions of Windows kernel components seen in the wild, we started to research the origins of these modifications. What we discovered are the artifacts of mitigations for well-known CPU side-channel vulnerabilities. In this article, we have a closer look at these potential attacks and how they are addressed by the Windows operating system. Finally, we discover and document the mechanics used in Windows executable files to help apply the mitigations, and point out that their understanding is crucial for an Anti-Cheat solution.

![Meltdown and Spectre vs Anti-Cheat](spectre-anticheat.png)

## Introduction
One of the tactics widely used by cheaters in multiplayer games is modifying the software loaded into memory by overwriting the original code of the process, or by injecting their own code. This can give them advantage in many ways -- they can modify the code of the game itself to overcome the obstacles that other players have to deal with (i.e. seeing enemies through walls or gain infinite amount of health). They can also modify third party software (e.g. the keyboard or mouse driver) in order to emulate input. One of the biggest challenges of anti-cheat software is to detect such modifications and to justify whether they are a cheat or just expected variations. But what happens if the operating system itself starts to modify the code while loading it into memory, and is doing it outwardly in a quite unpredictable way?

Few types of such intended modifications can be observed in Windows family systems since 2019 as a way of mitigating [Spectre and Meltdown](https://meltdownattack.com/) attacks. Those attacks, firstly described in 2018 are exploits that compromise the hardware security features of modern processors as they were at the time. Both of them are side channel attacks that, basing on modern CPU features, like out-of-order and speculative execution, making it possible for one processes to reach the memory reserved for another process (Spectre) or to the kernel memory itself (Meltdown).

## Background
But how are these attacks even possible? First of all we need to dig into the optimization techniques that are standard for modern CPUs.

### Pipelining
Every simple CPU instruction consists of the sequence of elementary operations, often 5 pipeline stages are assumed. First, the instruction needs to be fetched from the address pointed out by the instruction counter, then it needs to be decoded. After that, the actual arithmetical or logical operation may be executed (e.g. adding two numbers). The next step would be to write the result to the given register, and finally increment the instruction counter. To process each of the instructions sequentially would be a waste of time and resources - the actual arithmetical operation would be performed only during one of those steps, making the arithmetic logic unit (ALU) idle for the remaining four pipeline stages. Instead of the sequential execution of each instruction, processors reach for the next instruction as soon as the first step of the previous one was performed. That way many instructions may be performed in parallel, each one being at a different step of the pipeline at a given time. On modern CPUs there can be up to 100 instructions in flight in the pipeline.

### Out of order execution
As long as it's safe from the logical point of view, processors can switch the order of the instructions that are currently in the pipeline. Let's say that one of the instructions that is on the top of the pipeline needs to wait for data from the memory (not cached anywhere yet) in order to be finished. The process may execute the instructions that are later in the pipeline, but for which the data is already present, and move the execution of the previous instruction to the moment when the data will be available for it as well.

### Branch prediction
Usually the jump instructions are conditional and based on the result of the previous instruction. When both of these instructions are in the pipeline, there is no way for the processor to decide which instruction should be loaded to the memory next (the one after the jump is performed or the one at the jump target). Normally such situation would cause another delay in pipelining - until the moment when the result of the first instruction is known. Branch prediction technique tries to execute the more probable path based on the data from previous executions of that step. Yet the prediction may turn out to be wrong, which leads us to...

### Speculative execution
When proceeding with one or the other branch path, according to the decision made by branch prediction algorithm, the CPU makes a snapshot of the current state of itself. In the moment when the condition of the jump is known, processor can either continue with the execution (if the branch prediction was right) or discard all the instructions and restore the processor state according to the taken snapshot from before the jump (if the branch prediction was wrong). Speculative execution does not influence the state of the memory or disc - only the state of the CPU (its registers) can be changed. This way executing the wrong branch can be unrolled safely.

### Side channel attack
Spectre and Meltdown attacks are side channel attacks. It's a kind of attack that does not base on vulnerability of the algorithm, but instead tries to gain the confidential data using the meta information that can be gained from the technology used to implement the algorithm - in this case, the processor itself. One common examples of such attack is compromising comparing an input string with a password. If the algorithms compares one letter after another, the cheater is able to say how many first letters of the input string are matching the password, based on the time of the comparison – a timing side channel. Another kind of the side channel attack is any attack that tries to speculate over the content of the cache. By measuring the time for reaching a specific address in memory, the attacker is able to decide whether the content of that memory address is cached or not. 

## Performing the attack

### Meltdown
Meltdown is an attack that allows a user space process to reach the kernel address space content without any permissions to do so. Address space separation is assured at the hardware level – an instruction reaching for the kernel address space raises an exception that immediately prevents the data leak. In parallel to translating the address which the process want to reach, the processor checks the permission bits for that address to verify whether the process can have access to that address. If the permissions are insufficient, an exception will be thrown and the data won't be reached. This is perfectly safe for a processor that processes instructions sequentially and in given order. But due to pipelining and speculative execution, instructions that reach the forbidden kernel space are executed even though the user has no permission. Moreover, reaching the data and checking the permission are done independently. Of course the processor eventually will determine that such data requires higher permissions and will raise the exception -- but the data from kernel memory was already reached and put into one of the registers. Even though the processor state will be restored with the snapshot from before the execution of the feral instruction, the attacker can use the small window between getting the data and rejecting it, employing a cache attack.

### Spectre v1
Spectre does not break the user/kernel space barrier -- instead, it allows one process to reach the memory assigned to another process. This is possible by specific training the branch predictor in a way that it allows to execute (speculatively) the instruction that should never be reached in the regular flow. Common example is checking the index of the array before reaching the cell under that index. Even if the index is beyond the array size, and given the branch predictor is trained to execute the incorrect path, the data from outside the array will be reached, and for the moment will be available. To make use of that value before it will be rejected, in both Spectre and Meltdown, the cheater can use the probe array and timing attack to receive that value.

### Spectre v2
Second variant of the Spectre relies on exploiting the branch target predictor. Attacker may mistrain the branch target predictor in order to choose the indirect jump to the part of the memory where the malicious code is present. The code then access the sensitive data and buffers it in the way that it is possible to obtain it later. The rest of the attack is the same -- the speculative execution will be rejected, but the cache can be affected after that and using a cache attack, the attacker is able to obtain sensitive data.

### Cache attack
Even though the value from the forbidden part of the memory is reached and available in the register, it will be soon zeroed by the CPU at the moment when the speculative execution is rejected. To try to obtain this value, the attacker has to flush the cache totally, so that no previously stored data would affect the attack. Then the attacker can use the sensitive value as an index for the probe array, and reach for that index. The value from the probe array (for which the attacker has permissions) will be cached. Then, the attacker can perform a time attack on that cache and see for which of the indices the time of reading the value will be the shortest. This will reveal the forbidden value to the attacker.

## How to protect from those attacks?
There are several possibilities of mitigating the Spectre and meltdown attacks. Some of them were provided by the CPU manufacturers and were possible due to microcode updates. Depending on the type of the attack, a different approach was made. For example, for Spectre variant 1 sufficient solution is to add the [LFENCE](https://c9x.me/x86/html/file_module_x86_id_155.html) instruction. Such instruction stops speculative execution until the previous instructions are completed (retired). For meltdown, one of the possible solutions is Kernel Page Table Isolation ([KPTI](https://www.kernel.org/doc/html/latest/x86/pti.html)) (not in the scope of this article).

Yet for the Spectre variant 2 there is a huge performance problem. The solutions that were provided rely on limiting the branch target predictor. For example, indirect branch predictor barrier ([IBPB](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/indirect-branch-predictor-barrier.html)) flushes the content of the branch predictors which prevents the predictions based on previously ran software to influence the current one. Single Thread Indirect Branch Predictor ([STIBP](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/single-thread-indirect-branch-predictors.html)) restricts sharing the branch predictor data to a single physical core only. These approaches heavily impact the performance of the software, any may be unacceptable, especially in gaming industry. One of the most promising solutions is to implement the retpoline mechanism on the operating system level.

### What is retpoline?
Retpoline is a simple mechanism provided by Google engineers that allows to replace an indirect call or jump with a sequence that has a safe speculation behavior. A potentially unsafe indirect call is overwritten by a jump to that safe isolated space, followed by an interrupt (that breaks the speculative injection). In the isolated snippet below, the return address on the stack is overwritten by the desired address of the indirect call, followed by a return instruction.

So in the example 
([taken from 'Mitigating Spectre variant 2 with Retpoline on Windows'](https://techcommunity.microsoft.com/t5/windows-kernel-internals-blog/mitigating-spectre-variant-2-with-retpoline-on-windows/ba-p/295618)) for one of the types of retpoline code
```
call <Jump Target>
```

will be replaced by

```
RP0:  call RP2; push address of RP1 onto the stack and jump to RP2
RP1:  int 3   ; breakpoint to capture speculation
...
RP2:  mov [rsp], <Jump Target>; overwrite return address on the stack
```

to desired target

```
RP3:  ret; return
```


### How it's done
Retpoline changes are done ad-hoc when loading a kernel module to the memory. The question is how
does the OS accomplish these changes -- how does it know where the replacements need to be applied?
Where is all the information kept?

## Outlook
The answer is the Dynamic Value Relocation Table (DVRT), to be found via the *load config* directory of a Windows PE32 binary file. We will take a closer look in a [follow-up post](dvrt.md).
