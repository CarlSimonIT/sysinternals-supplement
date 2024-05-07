# Procexp Overview
Task Manager is a solid tool but it falls short on providing true insight into the root cause of a misbehaving process & outputting key data that would assist a technical user in identifying a process as malware. Comprehensively understanding the Windows operating system from the perspective of running processes requires the **ProcExp** Sysinternals utility.   
Upper pane of ProcExp displays tree view of processes. Lower pane displays either Handles (`CTRL+H`) or DLLs (`CTRL+D`)  
Use DLL view to drill down into the DLLs and mapped files loaded by the process.  
Use Handle view to inspect kernel objects currently opened by the selected process. These kernel objects can be of object type file, folder, registry key, window station, desktop, network endpoint, synchronization objects, and more.  
The Properties... dialog box of a process offers a wealth of information.  

## Measuring CPU Consumption
Older versions of Windows were only able to track an approximation of actual CPU usage. The period between clock-generated interrupts on most CPUs is 15.6 ms, which is the resolution that Windows identifies the thread currently executing on each CPU. If the thread is executing in kernel mode, its kernel-mode time is incremented by 15.6 ms. Otherwise, its user-mode time is incremented by that amount (?) The thread may have been executing for just a few CPU cycles when the interrupt fired, but the thread will be documented as occupying the entire 15.6 ms interval. Meanwhile, hundreds of other threads may have executed during that interval, but only the thread running when the CPU clock fired off the interrupt every 15.6 ms gets charged for that time.  
Task Manager will use the above approximations to report CPU usage even on newer versions of Windows. Accuracy is further reduced by rounding to the nearest 0.1% in the Process tab. Details tab rounds to nearest 1%.  
Processes with executing threads that consume very small amounts of CPU are indistinguishable in Task Manager from other processes that do not execute at all.  
> Prior to Windows 8, Task Manager did not account for CPU time spent servicing interrupts or deferred procedure calls (DPCs) and incorrectly included that CPU time in the System Idle Process.

A process that consumes very little CPU cycles is surprisingly taxing on system resources. There is a significant difference between a process that is not executing and a process that executes very frequently & briefly. A common bad practice is for developers to code for a process to periodically wake up to look for status changes. Instead, a process should take advantage of system-synchronization mechanisms that enable the process to not execute until an actual status change occurs. Every time a process is awoken and executes, it's code and data must be paged into the working set (memory) and possibly forces another range of virtual memory pages to be paged to disk (paged out.) Having the process wake up to check for status changes can also prevent a CPU from entering a more efficient power state, if that's important to you.  

By contrast, Procexp is a much stronger tool than Task Manager for several reasons. Procexp calculates process CPU usage from actual CPU cycles consumed rather than Windows' legacy estimation model in Task Manager. Procexp also conveys per-process CPU usage usage percentages to a resolution of 0.01% and does not display 0.00% for the aforementioned flickering processes. Instead Procexp notes CPU utilization as **<0.01**.  
CPU time spent servicing interrupts and deferred procedure calls is also cataloged separately and not lumped into the System Idle Process.  

>Process Explorer can accurately attribute all CPU cycles, including cycles for interrupts and deferred procedure calls. As such, when Procexp displays percentages the accounting is based on true CPU cycles, as opposed to Task Manager's inaccurate time-based accounting that rounds Process tab CPU to .1% and Details tab CPU to 1%. 

Procexp also illuminates other CPU usage measurements. For example, each thread tracks the number of times it underwent a context switch. The <span style="background-color:;color:yellow;font-family:verdana;font-size:1em;">Process Performance\Context Switch Delta</span> column will monitor and report changes in these numbers. Label is **CSwitch Delta**. 

A thread's _context_ is its set of machine registers, the kernel stack, a thread environment block, and a user stack in the address space of the thread's process... so I guess it's just the thread's "state."  
A _context switch_ is the suspension of a process/thread's execution & subsequent storing of its context so that the process/thread be restored and execution resumed later.  
Microsoft's documentation on context switches [here](https://docs.microsoft.com/en-us/windows/win32/procthread/context-switches) reads:
>The scheduler maintains a separate queue of executable threads for each individual priority level. These executable threads are known as _ready threads_. When a CPU core becomes available Windows performs a _context switch_. The steps in a context switch are:  
> 1. Save the context of the thread that just finished executing. Throw that in memory. Whatever. We just care that the thread's context is saved and available for retrieval later. RAM / disk isn't important.  
> 2. Place the thread that just finished executing at the end of the particular queue associated with that thread's priority. What's this about "placing" a thread? Uhhh what does it mean to move a thread? Does that even make sense? I now understand that they should've written _reschedule_ instead of place. The thread that just finished executing is being placed back in its priority-specific queue.  
> 3. The scheduler finds the highest priority queue containing a ready thread.
> 4. The scheduler removes that ready thread from the end of the queue, loads its context, and executes. 

"Mode transitions" between kernel mode / user mode are _**NOT**_ the same as context switches. Those are two completely different concepts. A mode transition could be a trigger for a context switch, but it is not itself a context switch. 

A context switch indicates that a thread has executed, but not how long it executed. For this, Windows tracks the actual quantity of kernel mode CPU cycles & user mode CPU cycles consumed by each thread. The <span style="background-color:;color:yellow;font-family:verdana;font-size:1em;">Process Performance\CPU Cycles Delta</span> column contains data on kernel mode / user mode CPU cycles for threads. Label is **Cycles Delta**.  
>So Cycles Delta refers to _mode transitions_. Very easy to forget! 

## Process Explorer & Administrative Rights
Process Explorer can run in a non-elevated security context but the data produced would be incomplete. In particular, Procexp will not display information about processes running in another LSA logon session.   
Instead of saying that Procexp should run with "Admin rights" it's more accurate to say that Procexp should run in the security context of an account that holds the **Debug Programs** privilege.  
>Remember that a user account that isn't a member of Administrators can have _effective_ administrative control over a computer if given the ability to configure or control software that runs in a more powerful security context. In the case of Process Explorer, a regular user that holds an "admin-equivalent" privilege like Debug Programs can use Procexp as if they were a member of Administrators.  

Domain environments can use Group Policy to withdraw the Debug Programs privilege from members of Administrators. So if a Sysinternals utility isn't working and you _know_ that your account is a member of Administrators you can look inside gpmc.msc.  
Processes running in a security context that holds the **Debug Programs** privilege are unable to modify **protected processes**. Even .\Administrator, the master account of master accounts on a Windows 10 machine, is unable to run software that can modify a **protected process**. Review the Application Isolation markdown file.

In addition to starting Procexp with Administrative rights the regular ways, Procexp has 3 additional pathways to running an elevated session:  
1. In a non-elevated Procexp window select File -> Show Details For All Processes. This restarts Procexp with the UAC prompt.  
2. Run Procexp with the /e switch:  <span style="background-color:Black;font-family:Consolas;font-size:1em;">C:\SysInt>Procexp.exe /e</span>
3. Toggle the Run At Logon feature to automatically start Procexp with elevated rights at next logon. The user account must be a member of Administrators. 