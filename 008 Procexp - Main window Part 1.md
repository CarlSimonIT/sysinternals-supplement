# Main window
The main window of Process Explorer has several components. Let's go through them. 

## Process list
Windows does not run processes. Windows runs _threads_. Threads, and not processes, are the entities that Windows schedules for execution and consume CPU clock cycles. A _process_ is merely a container for a set of resources, including at least one thread. It is not accurate to refer to an "active process" or a "process with running threads" because a process often contains no executing threads or even threads scheduled for execution.  
So each row in the process list actually represents an instance of an object of type _process_ with its own virtual memory address space, and with threads that may or may not be currently executing... and if not currently executing, may or may not be scheduled for execution... However... the first few rows of the process list are an exeption.  
>The book refers to rows in the process list as _running processes_. 

Row colors can describe the state of a running process or a type of running process. A _heatmap_ is a colored column that calls attention to a process' degree of resource consumption. [Default Color Selection settings](https://i.imgur.com/xinslFn.png).

Row color legend:  
* <span style="color:#D0D0FF;font-size:1em;">**Light Blue**</span> indicates an "own process" which is a process running in the same user account as the instance of **Procexp.exe** that's demarcating the "own process." NOTE: a process running in the same account as Procexp does **_NOT_** automatically mean that it's running the same LSA logon session, same integrity level, or same RDS session, and therefore not necessarily running in the same security context (?)   
* <span style="color:#FFD0D0;font-size:1em;">**Pink**</span> means the active process is a Windows service. Have a look under the process tree of wininit.exe -> services.exe. **lsass.exe**, **svchost.exe**, and **spoolsv.exe** are examples of processes get abstracted to a Windows service.
* <span style="color:#808080;font-size:1em;">**Dark Grey**</span> reflects a suspended process... which is a processes where all threads are suspended and cannot be scheduled for execution... Uhhh aren't those the same thing? Isn't a "suspended" thread the same as a thread that cannot be scheduled for execution (?)
  * The process (but not the Runtime Broker) of a UWP app enters Suspended status when minimized. 
  * A process that crashes (status of Not Responding) might appear as Suspended while Windows Error Reporting handles the crash. 
* <span style="color:#8000FF;font-size:1em;">**Violet**</span> denotes a "packed image" which is a program file containing executable code that's compressed, encrypted, or compressed & encrypted. Embedding executable code in a packed image is a common technique malware uses to evade anti-malware. Once the file with the packed image is on disk it unpacks itself into memory and executes. NOTE: Procexp uses simple heuristics to identify packed images so false positives sometimes result. 
* <span style="color:#D06C00;font-size:1em;">**Brown**</span> represents a process that is associated with an instance of a _job_ object. Remember: a job object is a Windows construct that allows >=1 processes to be managed as a unit. Jobs can be bound to occupy a maximum quantity of memory, a limit on execution time duration, or other constraints on resource usage. A process may not be associated with >1 job. Job objects in Procexp are not highlighted by default.  
* <span style="color:#FFFFA0;font-size:1em;">**Yellow**</span> highlights .NET processes which are processes that use the Microsoft .NET Framework. .NET Processes in Procexp are not highlightedby default.  
* <span style="color:#00EAEA;font-size:1em;">**Cyan**</span> entries are "immersive" processes which just means the process of a Universal Windows Platform app. Under the surface the _IsImmersiveProcess_ API function is what determines whether a process is "immersive." Weirdly enough, the **explorer.exe** process uses the regular Win32 API but is marked as immersive in Procexp. This is because it renders the modern Start menu and that's good enough of an explanation for me.  
* <span style="color:#FF0080;font-size:1em;">**Bright Pink**</span> symbolizes a **_protected process_**. Protected processes in Procexp are not highlighted by default. 

The order of precedence is Suspended, Immersive, Protected, Packed, .NET, Jobs, Services, and Own Process. 

>NOTE: An instance of Procexp running in a non-elevated security context will not produce the color category for (1) a running process that contains a packed image, (2) a running .NET process, or (3) a running process that's associated with a Job Object... if the integrity level of said process is higher than that of the Procexp.exe process ... or if said process is in a different user account from our instance of Procexp.exe.

Procexp temporarily highlights new processes in <span style="color:#46FF46;font-size:1em;">**Green**</span> and recently exited processes in <span style="color:#FF4646;font-size:1em;">**Red**</span>. Default is 1 sec but it's a good idea to bump it up to 3 under Options -> Difference Highlight Duration.  
These green and red color codes are also found in the Handle view (CTRL+H) and the DLL view (CTRL+D).  
The <span style="color:#FFFFA0;font-size:1em;">**Relocated DLLs**</span> color category is specific to the DLL view and not applicable to the process list. 

Procexp updates the values of dynamic attributes that appear on screen at an automatic refresh interval of once per second. A dynamic attribute is just a variable that's likely to change often. Examples include Cycle Time, CPU %, and CSwitch Time. Pause/unpause this updating by pressing **SPACE**. Force a refresh of all static & dynamic attributes with **F5**. Change the automatic refresh interval through View -> Update Speed. Keep in mind that the value of a static attribute that appears on screen can only be trusted after hitting F5! 

Columns. Process, CPU, PID, Description, Company Name, etc. are obvious.

* _Private bytes_ is the # of bytes allocated and commited by the process for its own use and that are not sharable with other processes. Per-process private bytes include heap and stack memory (?) A continuously increasing value of Private Bytes is symptomatic of a memory leak.  
* _Working Set_ is the amount of physical memory assigned to the process by the Virtual Memory Manager subsystem of the Windows Executive. The quantity of memory allocated to a process' Working Set = Private bytes + Sharable bytes.  

>Refer to [this article on physical and virtual memory in Windows 10](https://answers.microsoft.com/en-us/windows/forum/windows_10-performance/physical-and-virtual-memory-in-windows-10/e36fb5bc-9ac8-49af-951c-e7d39b979938)

Process Tree. 3 views of the Process column: ascending, descending, and Tree which lays out the parent/child relationship between processes. REMEMBER! A child process holds the PID of its parent. However Windows never uses that info for anything. Unlike UNIX-based operating systems, Windows does not use the parent/child relationship of processes. The PID of a parent that a child holds in memory is _not_ updated to identify another ancestor process. Tree view left-aligns processes with no existing parent. Discord.exe, for example.

Tooltips. Hover the mouse over everything you can. You'll uncover quite a bit of helpful information. 

### System processes, Startup/Logon processes, and User processes
---
**System processes**. System Idle Process, System, and Interrupts are the rows of focus here. The System Idle Process and Interrupts **_are not real operating system processes_**. Details forthcoming. The System process **_does not run user mode code_**. Details forthcoming.  
* System Idle Process has 1 "thread" per CPU and is used to account for CPU idle time when Windows is not running any program code. Even though it's assigned PID 0, it's not a real process. There is no PID 0 in Windows. That static attribute of the System Idle Process holding PID 0 is simply an on-screen entity created by Procexp and represents no underlying data structure. Furthermore, displaying System Idle Process as if it were any other _running process_ is simply a convention taken from Task Manager for the purpose of convenience. It's best to describe System Idle Process as a _pseudo-process_.   
* System process hosts only kernel mode system threads that run in kernel mode and never user mode. Threads within the System process typically execute code within **ntoskrnl.exe** and device driver code. System process & threads always execute code in kernel mode.  
* Interrupts is a pseudo-process that represents a CPU's time spent in kernel mode and dedicated to servicing interrupts (aka hardware interrupts or CPU interrupts) & deferred procedure calls. The Interrupts row is represented as a child of System because its time is spent entirely in kernel mode but you must never, ever forget that the Interrupts row is _**NOT**_ a process. Unlike Task Manager, CPU time occupied by servicing interrupts & DPCs is not charged to the System Idle Process. 

>ASIDE: a system suffering high interrupt or DPC load would benefit from troubleshooting with the [Xperf](https://docs.microsoft.com/en-us/windows-hardware/test/wpt/xperf-command-line-reference) utility for tracing interrupts & DPCs and the [Kernrate Viewer](https://www.microsoft.com/en-us/download/details.aspx?id=24853) to monitor kernel mode CPU usage. _Windows Internals_ provides a thorough treatment of interrupts and DPCs. 

**Startup / Logon processes**. Power up your machine. From the time the UEFI transfers control to software, to the moment the first user logs on there's a well-defined sequence of processes. Many of these processes have exited by the time you're able to inspect the process tree. _Windows Internals_ provides a thorough treatment of the Windows NT startup process. Here is a very brief and incomplete overview of the steps prior to starting an interactive user session:   
The System process **ntoskrnl.exe** creates an instance of the Session Manager subsystem **smss.exe** process which stays running until system shutdown. The Session Manager subsystem launches 2 new instances of itself: one in _session 0_ and one in _session 1_. Each instance of smss.exe creates processes in their respective sessions:
* _Session 0_ instance of smss.exe creates an instance of the Client Server Runtime Process **csrss.exe** and an instance of the Windows Start-Up Application **wininit.exe**. From there, wininit.exe launches an instance of the Service Control Manager **services.exe** and an instance of the Local Security Authority subsystem **lsass.exe**. NOTE: most Windows services are just child processes of the session 0 instance of services.exe (?) The session 0 instance of services.exe does not host any services itself.  
* _Session 1_ instance of smss.exe establishes an instance of csrss.exe & an instance of the Windows Logon Application **winlogon.exe** process. winlogon.exe launches an instance of the Windows Logon User Interface **logonui.exe** process (which runs in the Winlogon desktop of the Winsta0 window station of RDS session 1!) logonui.exe collects un/pw from keyboard input and if authentication succeeds winlogon.exe creates an instance of the Userinit Logon Application **userinit.exe** process which in turn kicks off the Windows Explorer **explorer.exe** process that's tasked with providing an interactive GUI. logonui.exe & userinit.exe exit by the time the interactive user session starts. 

>>>CONFUSION: The book reads that smss.exe creates additional instances of itself in session 0 and session 1, and that these instances of smss.exe exit before the user first logs on. This is why is why the instance of smss.exe we find in Procexp reflects no child processes: it's the original instance of the Session Manager subsystem spun up from ntoskrnl.exe. Fine. I get that. But then which RDS session does that instance of smss.exe displayed in Process Explorer occupy? (?) Probably session 0. That's the only answer that makes sense.   

Process Monitor `Procmon.exe` has boot logging features that reveal the complete startup process. 

**User processes**. Certain seemingly unusual patterns in user processes arise regularly. Let's go through the some of the most representative instances where Procexp doesn't immediately make sense.   
* **Typical pattern 1**: A user process is not necessarily a child process of explorer.exe.
  * I'm running Procexp as RYZEN3900X\CarlS and the instance of the CTF Loader **ctfmon.exe** process is listed as an Own Process. So ctfmon.exe is running in the security context of my user account. What's weird is that ctfmon.exe is shown to be a child of **svchost.exe** which is a process that hosts a Windows service, and not a child of explorer.exe. The most common explanation for these situations involves an out-of-process DCOM component: an app invokes a component that COM determines needs to be hosted in a new separate process. Say that component is the Collaborative Translation Framework, and COM determines that this new separate process is ctfmon.exe. The parent process that launches the ctfmon.exe instance is not the client app. Instead, the parent process is **C:\Windows\system32\svchost.exe -k DcomLaunch -p** which is the process that hosts the DcomLaunch service. ctfmon.exe runs in the security context of the interactive user so it's considered an Own Process, and it's not a child of explorer.exe.  
  * Another example (in Windows Vista & Windows 7) of an Own Process that is not a descendant of explorer.exe is the Desktop Window Manager **dwm.exe** that runs in the context of the local user and is launched by the process **C:\Windows\System32\svchost.exe -k LocalSystemNetworkRestricted** that hosts the UxSms service.  
  * On Windows 8 and beyond dwm.exe runs in the security context of a system-managed Window Manager account and is started by winlogon.exe.

* **Typical pattern 2**: Processes under the control of a job object. 
  * Some DCOM components, particularly Windows Management Instrumentation (WMI) hosting processes, run with restrictions like the amount of memory they can allocate, the number of child processes they can spin up, or a cap on the amount of CPU time they can charge. Such boundaries are promulgated by a job object. And any process launched through the Secondary Logon service (seclogon) (executable:**C:\Windows\system32\svchost.exe -k netsvcs -p**) is added to a job object so that the launched process & child processes are tracked as a unit and terminated if they're still running when the user logs off. 
  * The Program Compatibility Assistant (PCA) service tracks legacy apps so that it can offer a compatibility fix if it detects a potential compatibility problem for which it might have a solution after the last process in the job object exited. 
>Launching an app with the **RunAs** parameter is an example of using the Secondary Logon service.  
>><span style="background-color:Black;font-family:Consolas;font-size:1em;">Start-Process PowerShell -Verb **RunAs**</span>
>
> Ugh. running this doesn't produce a job object. Whatever. 

Virtualization-based security in Windows 10 and Windows Server 2016/2019 can enable features like Device Guard and Credential Guard, and can launch user mode processes that are outside the direct control of the bare-metal OS. Procexp can confirm the existence of Secure System and LSA isolated **lsaiso.exe** but very little information beyond that. 

### Process actions
---
Select a running process then the "Process" in the menu. Many options abound. Probably 20% are worth looking at.  
**Set Affinity** will prevent an instance of a process from running on CPU cores you designate. Useful for a runaway process that occupies all CPU clock cycles, but needs to be stay running yet throttled for troubleshooting. doomseeker.exe starts zandronum.exe. Withdrawing CPU affinity for CPU0-3 keeps the process running and Task Manager reveals that different cores are being used. 

> Permanently restricting a process to subset of CPU cores requires the **SingleProcAffinity** application compatibility shim from the [Application Compatibility Toolkit](https://techcommunity.microsoft.com/t5/ask-the-performance-team/demystifying-shims-or-using-the-app-compat-toolkit-to-make-your/ba-p/374947). An alternative is modifying the app's source code, and a last resort is modifying the file's PE header to special affinity.

**Set Priority** adjusts the base scheduling priority for the process. doomseeker.exe & zandronum.exe are both set to Normal. 

**Restart** Procexp terminates the instance of the process and restarts with the same image using the same command-line arguments. NOTE: the new instance might fail to work correctly if the original process instance depended on other operating characteristics like security context, environment variables, or inherited object handles.  
Hit Restart on zandronum.exe as an experiment. The instance exits as expected but reopens as a child of Procexp64.exe and not doomseeker.exe! 

![](https://i.imgur.com/PJfUINi.png)  
zandronum.exe starts with all settings reset to default because user configs are inherited from doomseeker.exe. It's as if Procexp.exe ran the command <span style="background-color:Black;font-family:Consolas;font-size:1em;"> C:\>zandronum.exe -connect 104.128.49.2:10674 -iwad doom.wad -file brutalv21.pk3</span>. Can confirm **excel.exe** also starts as a child of Procexp64.exe. 

**Suspend/Resume** You already understand this. There's Running, Not Responding, and Suspended. Suspend a process to free CPU, disk, network resources.  
>Suspend/Resume is useful for "buddy system" malware. This variety of malware features >=2 processes in which a nonterminating process launches the other process exited. Defeat such malware by suspending the processes first and then terminating. 

**Launch Depends** Not going to get too deep into this. If the Dependency Walker utility (Depends.exe) is found Procexp launches it with the pass of the executable image of the selected process as a command-line argument. Depends.exe shows DLL dependencies. NOTE: Dependency Walker is no longer in active development. Instead you could try using [Dependencies](https://github.com/lucasg/Dependencies). Don't even know if this is a feature of Process Explorer anymore (?)

**Debug** Must first register a debugger in `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AeDebug`. Registering a debugger means specifying the default postmortem debugger. [Reference](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/enabling-postmortem-debugging#configuring-post-mortem-debuggers). Download and install the Debugging Tools for Windows & in cmd.exe perform:
```CMD
C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe –I
C:\Program Files (x86)\Windows Kits\10\Debuggers\x86\windbg.exe –I
```
Restart Procexp.exe and the Debug menu item appears. See _Windows Internals_ for further reading. 

**Create Dump** for either a minidump for a full memory dump in a .dmp file for the selected process.  
**Check VirusTotal** submits the SHA1 hash of the process' image file to VirusTotal.com and outputs result to VirusTotal column.  

## Column selections to display process attributes
In the Handles table, DLLs table, and process list you can right-click Select Columns and view a wealth of information. 100+ columns available for processes and 40+ for Handle & DLL tables. 


