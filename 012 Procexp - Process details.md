# Process details
Here's some vocab for you. A dialog box of an app can either have a _mode_ or be _modeless_. The Options dialog box of Word & Excel has a mode. The dialog box of the Properties windows of Process Explorer are modeless.  

## Image
[Image tab example](https://i.imgur.com/60Qfn5g.png). Most of the info is static for the lifetime of the process. 
## Performance
[Performance tab example](https://i.imgur.com/Qm87atm.png). Notice that CPU time is split into Kernel Time and User Time. There's no other way to precisely quantify in Process Explorer, that I can see. 
## Performance Graph
[Performance Graph tab example](https://i.imgur.com/lVGEv2H.png). I/O is split out into R+O and W. The Private Bytes signify the amount of System Commit, which to me is still unclear (?) For now understand that continual, uncontrolled growth in the Private Bytes graph indicates a memory leak. 
## GPU Graph
[GPU Graph tab example](https://i.imgur.com/pB46Tc4.png). By default, Procexp's GPU usage calculations reflect only 1 GPU engine of one GPU. Refer to the **System Information** section on how to select engines for inclusion in these calculations. 

<span style="font-family:Papyrus; font-size:2.1em;">Place both Sapphire GPU's in your system but don't set up CrossFire. See how Process Explorer reacts. Then compare post-config of CrossFire.</span>  

## Threads
[Example of Threads tab](https://i.imgur.com/geF2Nls.png). 

## TCP/IP
Now, this is useful. 
[Example of TCP/IP tab](https://i.imgur.com/T58tpeO.png)

## Security
The _process token_ defines the security context for the process. And the security context of a process is defined as the user principal the process is running as, the user's group membership, and that user's systemwide privileges. Have a look at the [Security tab of the process that runs Minecraft](https://i.imgur.com/cAoJE2n.png). The header reads the RDS session in which the process runs, the process token's _**LSA logon ID**_ which looks like a61a9. 

The Group column signifies the groups that the process owner account has membership in. View the <span style="font-family:Verdana;">**SID**</span> of that security group at the bottom of the list. 

In must circumstances, particularly desktop apps, the **Windows Security Reference Monitor** performs access checks with the process token. In some cases it performs the access check (when a process requests access to a resource) with a thread token derived from the process token. This thread token will never have more rights than the process token. Info on the Security tab can shed light on the success or failure of an operation. 

Services & server apps can impersonate the security context of a different user when performing actions on behalf of that user. For our examples, let's call this UserA. Impersonation is implemented by associating a copy of UserA's token with a thread within the process. During impersonation, access checks are performed with the thread token, so in these cases the process token might not be applicable. The dialog box does not show thread tokens. 

Several main ideas on security tokens are:  
* A group with the Deny flag set can effectively be considered to not be present in the token at all. With UAC, powerful groups such as Administrators are marked Deny-Only except in elevated processes. The Deny flag indicates taht if an object has an "access-allowed" access control entry (ACE) for Administrators in its permissions, that entry is ignored, but if it has an "access-denied" ACE for Administrators (which is uncommon), the access is denied.  (?)  This whole bullet point was confusing to me.     
* The privilege Disabled is very different from a privilege not being present. If a privilege is in the token, the program can enable the privilege and then use it. A privilege being absent means the process cannot acquire it. Note that several privileges are considered administrator-equivalent like Debug. Windows places a hard block on allowing these privileges to appear in the security token of a standard user.    
* If a domain computer cannot contact a DC, and has not cached the results of the previous SID-to-name lookups, it cannot translate the SIDs for groups into the group names. In this case, Process Explorer simply displays the SIDs. Also in the case of a computer that used to be domain-joined, right?   
* The group called **Logon SID** (?) is based on a random number generated at the time the user logged on. One of its uses is to grant access to terminal server session-specific resources. Logon SIDs follow the format of S-1-5-5-**X**-**Y**.     
For example, we had the account `NT AUTHORITY\LogonSessionId_0_69475268` which was assigned a logon SID value of S-1-5-5-**0**-**69475268**. 

At the very bottom is the Permissions button. This contains the security descriptor for the process instance itself. That is, which security principals can perform which action.  
.\CarlS starting Minecraft not as an admin   
https://i.imgur.com/YpKJvHw.png  
https://i.imgur.com/FoOGcqM.png  

.\CarlS starting Minecraft as an Admin:  
https://i.imgur.com/1kswkrO.png  
https://i.imgur.com/gYXkv12.png  

Try this on the nonadmin account. 

## Environment
Simply lists the process' environment variables & corresponding values. Didn't know that environment variables were process-specific. [Example of Environment tab](https://i.imgur.com/Hvxbzmz.png). A child process usually inherits the environment variables from the parent. Often the environment blocks of all processes will be substantially equivalent. Exceptions include:  
a. a parent process that's able to specify a different set of environment variables for a child process 
b. if each process can add/delete/modify its own environment variables and 
c. Windows might broadcast a message that alerts running processes that the environment variable configuration for the system has changed. It's not guaranteed that all processes will receive the notification, console programs in particular. And even if a process receives a message, it's not guaranteed that the process will update its own environment block with the new settings.

## Strings
Simply the list of all sequences of 3 or more printable characters found in the image file of the process. Select the Image or Memory radio button to view the set that's on disk or in-memory, respectively.  
>NOTE: Process Explorer does not uncover strings found in the entire scope of committed memory a process instance's virtual address space. Process Explorer only inspects the region of virtual address space where the executable is mapped. 

The strings within the image and memory may differ when the image is decompressed. Or the image can be decrypted when loaded into memory, resulting in a difference between the series of 3+ strings. 

Yikes, memory strings can also include dynamically constructed data areas of the image's memory range. Like if the process instance writes a timestamp into a page of virtual memory address space where the executable is mapped... or if the process instance pulls code from online and writes into that region of address space.  

## Services
Windows services usually run in a noninteractive process that can be configured to start independently of any logged on user. Services are controlled through a standard UI with **services.exe**, the Service Control Manager. Multiple services can be configured to share a single process, the most representative process being **svchost.exe** which is specifically designed to host multiple services implemented in separate DLLs. 

Remember that a Services tab only appears when the process hosts a service. 

A description of the highlighted service appears at the bottom. [Example of a Services tab](https://i.imgur.com/oRzyWm5.png). 

Individual services can be configured to allow or prohibit stop/pause/resume/etc. operations. Process Explorer enables Stop, Restart, Pause, and Resume buttons if the service allows those operations. Our example does not allow Pause or Resume. 

The Permissions button leads to the security editor dialog box for that Windows service. One may permanently change the permissions of a service, probably shouldn't though. [Rights and privileges specific to Windows Services](https://i.imgur.com/8bSxE92.png) include Query Status, Interrogate, User-Defined Control, and Stop. Standard privileges like Read Permissions, Change Permissions, and Change Owner are also present.  

>NOTE: Granting a non-administrator the Write permission (a general right found in all security dialog boxes) or the Change Config, Change Permissions, or Change Owner (again, specific to Windows services) permission for _any_ service opens a privilege elevation exploit that makes it easy for the non-admin to take full administrative control over the computer. 

## .NET
who cares

## Job
A job object allows groups of processes to be managed as a unit, and to enforce constraints on the associated processes. A job can enforce different values to limit maximum Working Set, CPU rate, I/O rate, or process priority on a per-process basis... or for all processes associated with the job as a whole. Jobs can specify CPU core affinity, prevent access to the clipboard, or terminate all processes at once. [Example of a Job tab](https://i.imgur.com/ymBAhaj.png). Listed is the name of the job (if one is defined), the processes associated with the job, and the limitations placed on each process. Your "Call Stacks and Symbols 

## Threads 
Remember that _a process does not run code_. Only **_THREADS_** run code. A thread's resources include a call stack and an instruction pointer that identifies the next executable instruction. Your "Call Stacks and Symbols" markdown document is a good reference.  
* **TID** is just thread ID. A system-assigned unique thread identifier. You already know this.    
* **CPU** The % of total CPU time, across all cores, that the thread was executing during the previous refresh cycle. This value maxes out a 1/#CPU cores. For my system with 24 logical processors the max value is 1/24 which is 4.16666%. Filtering out CPU cores with "processor affinity" settings does not appear to have an effect.      
* **Cycles Delta or CSwitch Delta** If Procexp is running a context that gives full control over the process then the Threads tab will display the # of CPU cycles that thread consumed since the last refresh. Namely, the Cycles Delta column will be present. However, if run in a less-privileged security context then Context Switch Delta is provided. Same for protected processes. **Context Switch Delta** is the number of times that the thread has been given control, and has begun executing since the previous update. A thread can perform a context switch hundreds of times per second. [Example of Context Switch Delta](https://i.imgur.com/cJzAh5r.png).    
* **Service** for processes hosting a service the Service column displays which service a thread is associated with. [Example of Services column in Thread tab](https://i.imgur.com/o2wdvNc.png). Windows tags the threads of service processes to associate threads (and TCP/IP endpoints) with their owning service.     
* **Start Address** Remember the format of a call stack: _module_!_function_+_offset_. Start Address is the symbolic name associated with the program-specified location in the process' virtual memory where the thread began executing. If Process Explorer is configured to use a symbol server, displaying this column might introduce lag as required symbol files are downloaded. An indicator appears above the list box when this is happening.    
[Example of Thread tab for process running DHCPv6 client](https://i.imgur.com/LfwOC1h.png). When the Windows API function _Dhcpv6Main_ is running the call stack looks like the image. 

At the bottom, one can also observer the amount of Kernel time & User time spent on the process, total # of context switches, charged CPU cycles, etc. [Zandronum](https://i.imgur.com/Da1fMud.png) produces call stacks that are opaque to me at this point. 