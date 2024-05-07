# Processes, Threads, and Jobs
A _program_ is a static sequence of instructions. Your batch file to generate random numbers is a program. A _process_ is a container for a set of resources used to execute the instance of a program. At the highest level of abstraction a Windows process comprises of the the following:  
1. A unique identifier called a Process ID (PID). There are currently 194 processes running on my system. A PID is internally part of an identifier called a _client ID_. 
2. At least one _thread_ of execution. Every thread in a process has full access to all the resources referenced by the process container. There are currently appx 2480 active threads. It's possible to have a process with no threads but that's uncommon. 
3. A _private virtual address space_ which is a set of virtual memory addresses that the process can use to store and reference data and code. **Virtual Address Descriptors** (**VADs**) are data structures that the Virtual Memory Manager subsystem uses to keep track of which block of virtual memory a process is using. 
4. An executable program, which defines initial code & data and is mapped into the process' virtual address space. 
5. A list of open _handles_. A handle is an entry in a table that maps to various system resources that are all accessible to the threads in the process. Such resources include kernel objects like files, communication ports, and shared memory sections... or synchronization objects like mutexes, events, and semaphores. A process has a "handle table." Task Manager shows there are 84148 "handles." Here does "handle" = "open handle?" Start Task Manager -> Details tab -> show Handles column. Displayed are the number of handles to kernel objects opened by threads running within the process. Key takeaway is that a _handle is a reference to an instance of an object_. 
6. A security context which is a data structure called an _access token_ that identifies the user, security groups, privileges, attributes, claims, capabilities, UAC virtualization state, LSA logon session ID, remote desktop services session ID, and limited user account state associated with the process. The access token also holds the AppContainer identifier and the AppContainer's related sandboxing information. 

A process holds a record of the PID of its parent process. However, if the parent process exits, the record of what that PID is does not update. A process may reference a nonexistent parent or even a completely different process that happens to have been assigned that same PID. This recording of ther parent PID is for informational purposes. 

## Tools for Viewing and Modifying Processes
Examples are the _Debugging Tools for Windows_ from the [Windows SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk/), the Sysinternals tools, and Task Manager! Realize there's no such thing as a 'task', _per se_ in the Windows kernel.

Start Task Manager -> Details tab -> show Handles column. Displayed are the number of handles to kernel objects opened by threads running within the process. Now show Status column. There are **three** types of Status:

1. Running. For processes with a UI Running means 'waiting for input' 
2. Suspended if all the threads in the process are in a suspended state. For UWP apps it means the window has lost its foreground status and is minimized by the user.     
3. Not Responding. A proc will fall into (Not Responding) if the thread that created the UI has not checked its message queue for UI-related activity for at least 5 seconds. The thread in the process that owns the UI window may be doing something CPU-intensive, or waiting for an I/O operation to complete or whatever. Either way, the UI window fades and freezes.    

Each process points to the PID of its parent process. As an example, run this in cmd:
>wmic process get Name, ParentProcessID, ProcessID

The parent process can be the creator process, but not always. In certain cases, some processes that appear to be created by a certain user application might involve the help of a broker process (aka a helper process) which is responsible for calling the process creation API. In such cases, it would be confusing (and sometimes incorrect, if handle or address space inheritance is needed) to display the broker process as the creator, and a "re-parenting" is done. The complicated details are found in Chapter 7 of Windows Internals.

If the parent process no longer exists leaving the child process still running then that ParentProcessID value is still not updated. It's possible for a child process to refer to a nonexistent parent process in the form of an old ParentProcessID value.  

## Threads
A process is merely a container and technically it's not a process that runs but the process' _threads_. A "thread" is the entity within a process that Windows schedules for execution. A thread includes the following essential components:  
1. A unique identifier called a _thread ID_. A thread ID is part of an internal structure called a _client ID_. Values for Process IDs and thread IDs are generated out of the same namespace so they never overlap.    
2. A private storage area, whatever that means, called _thread-local storage_ (TLS) for use by subsystems, run-time libraries, and dynamic-link libraries (DLLs.)  
3. The contents of a set of CPU registers that represent the state of the processor. (?) By that do we mean kernel mode vs. user mode(?) These registers include an instruction pointer that identifies the next machine instruction the thread will execute. (?)   
4. Two _stacks_, one for the thread to use while executing in kernel mode and one for the thread to use while executing in user mode.(?)  
5. Threads can, but not always, have their own _security context_ (aka _execution context_) that is often used by multithreaded server applications that impersonate the security context of the clients they serve. This security context is aka a _token_. A thread token is often used by server apps to impersonate the security context of the clients they serve. So that's a thread's "security context."

Every thread in a process shares that process' virtual address space. Remember that data and code are also stored in this virtual address space. So data structures used by a thread are not protected from being read or modified by other threads in the same process. Each thread in a process has full read-write access to that process' resources and virtual address space.

Threads are unable to reference the address space of another process by default. However **[1]** the other process may demarcate a range of its private virtual address space as a _shared memory section_ (called a **file mapping object** in the Windows API) and allow a foreign thread to access that range of address space. **[2]** Another process may have the right to open another process to use cross-process memory functions in the Windows API like **ReadProcessMemory** and **WriteProcessMemory**. By default within a single user account, Process A has the right to open Process B and run ReadProcessMemory/WriteProcessMemory. However Process B could have certain protections against being targeted by these Windows API functions. Process B could also reside in an AppContainer or other sandbox and be shielded from having Process A run these Windows API functions on it. 

Threads do not have an access token by default but can obtain one. A thread assigned an access token can impersonate a different security context, even the security context of a process running on a remote system. Other threads in the process would be unaffected by the assignent of an access token to this thread. 

A thread's volatile and non-volatile CPU registers & private storage area are known as the thread's "context." So when asked "What's the thread's context?" the correct answer will include the associated CPU registers and thread-local storage. This info is different for each architecture that Windows runs on. Use the **GetThreadContext** Windows API function to access this architecture-specific information (aka the **CONTEXT** block.) Additionally, each thread has its own stack 

Threads have their own execution context. Threads in a process also share the process' virtual address space & associated resources belonging to the process. Therefore, all threads in a process have full read-write access to the process' virtual address space. 

There are 2 cases where a thread will access the address space of a foreign process. 

## Jobs
Windows provides an extension to the process model called a _job_. The main function of a job object is to allow a group of processes to be managed as a unit. For example, a job can terminate a group of processes all at once instead of one at a time. The calling process does not need to know which other processes are in the group.

Job objects also allow control of certain attributes and provides limits (?) for the process or processes associated with the job. e.g. Jobs can enforce per-process or job-wide limits on user-mode execution time and committed virtual memory. Book provides an example of how the Windows Management Instrumentation functions: WMI loads its providers into separate host processes controlled by a _job_ that limits memory consumption & the total number of WMI provider host processes that can run at one time.

Job objects record basic accounting info for all processes associated with that job object, including processes that have been terminated. In a way, job objects compensate for the lack of a structured process tree in Windows. Process Explorer can show which processes a job object manages.  

>>>FROM 08: a job object is a Windows construct that allows >=1 processes to be managed as a unit. Jobs can be bound to occupy a maximum quantity of memory, a limit on execution time duration, or other constraints on resource usage. A process may not be associated with >1 job. Job objects in Procexp are not highlighted by default.  

## Fibers vs. User-mode scheduling (UMS) threads. 
Switching execution from one thread to another involves the kernel scheduler and is computationally expensive. Windows implements _fibers_ and _UMS threads_ to reduce the burden.  
>NOTE: threads of a 32-bit app running on 64-bit will contain both 32 & 64-bit contexts, which WoW64 will use to switch the app from runing in 32-bit to 64-bit mode when required. These threads have two user stacks and two CONTEXT blocks, and the usual Windows API functions will return the 64-bit context instead. The **WoW64GetThreadContext** function, however, will return the 32-bit context. Chapter 8 has the details. 

### Fibers
Fibers allow an app to schedule its own threads of execution instead of the priority-based scheduling mechanism built into Windows. Fibers are often called _lightweight_ threads. The scheduling activity of fibers is invisible to the kernel because they're implemented in user mode in Kernel32.dll. To use fibers you must first call the Windows **ConvertThreadToFiber** function which converts the thread to a running fiber. From there the fiber can create additional fibers with the Windows **CreateFiber** function (a fiber can have its own set of fibers.) Finally, for the fiber to begin execution it must be manually selected through a call to the Windows **SwitchToFiber** function. The new fiber runs until it exits, or until it calls SwitchToFiber and selects another fiber to run. Look up the Windows SDK documentation for the whole story. 

A fiber is not an entity unto itself like a thread. It's meaningless to say that a thread was "converted" to a fiber. It's more accurate to say that a thread is under the influence of a fiber. But for simplicity we'll use the term **_fiber_**.

>Converting threads to fibers in order to bypass the kernel scheduler is generally bad practice. This is because fibers are invisible to the kernel. Fibers also have issues sharing thread-local storage because it's possible for several fibers to run on the same thread. Fiber-local storage (FLS) exists, but it does not solve all sharing issues, and I/O-bound fibers will perform poorly regardless. Additionally, fibers cannot run concurrently on more than one processor, and are limited to cooperative multi-tasking only. In most scenarios it's best to let the Windows kernel schedule threads for the task at hand.  

### User-mode Scheduling (UMS) threads. 
Only for 64-bit versions of Windows. UMS threads have their own kernel thread state and are therefore visible to the kernel. Unlike fiber-controlled threads, UMS threads can issue blocking system calls to share & content for resources. Or, when 2 or more UMS threads need to perform work in user mode, they can periodically switch execution contexts (by yielding from one thread to another) in user mode rather than involving the scheduler. From the kernel's perspective the same kernel thread is still running and nothing changed when when one UMS thread yielded to the other.

When a UMS thread performs an operation that requires entering the kernel (such as a system call) the UMS thread switches to its dedicated kernel-mode thread, called a _directed context switch_. Concurrent UMS threads cannot run on multiple processors. However they do follow a pre-emptible model that's not solely cooperative.

## Virtual Memory
The virtual memory system of Windows is a flat non-hierarchical address space that provides each process with the illusion of having its on vast private virtual address space. Virtual memory provides a logical view of memory that might not correspond to its physical layout across volatile RAM & disk. At run time the Virtual Memory Manager subsystem maps virtual memory addresses onto physical memory addresses and keeps Process A from overwriting data of Process B. Both RAM and virtual memory are quantized in the form of _pages_. Windows sets the default page size as 4 KB. 

The Virtual Memory Manager subsystem will **_page_** virtual memory to disk when the quantity of virtual memory outstrips the quantity of physical memory. This range of virtual memory mapped to disk is represented as the **paging file** _pagefile.sys_. Paging data to disk frees physical memory so that it can be used for other processes or for the OS itself. When a thread accesses a range of virtual memory that has been paged to disk the Virtual Memory Manager loads the data from disk back into RAM. 

The function of paging is transparent to processes and threads. Processes and threads are unaware of whether a range of virtual memory has been paged to disk. The Virtual Memory Manager subsystem, with support from the underlying hardware, will load the appropriate data from disk into RAM when a thread calls on a range of virtual memory.

Theoretical max of virtual memory on 32-bit x86 systems is 4 GB. Windows reserves addresses 0x00000000 through 0x7FFFFFFF (lower half for virtual memory address space) to the private virtual address space of processes. Addresses 0x80000000 through 0xFFFFFFFF is allocated for Windows' own protected OS memory utilization. The lower 2 GB can be considered 'user process' address space with the upper 2 GB being considered 'system' address space. 
> ASIDE: Windows supports boot-time options like the **increaseuserva** qualifier in the BCD that give processes running flagged programs the ability to use up to 3 GB of private address space, leaving 1 GB for Windows. This 'flag' is the large address space-aware flag that must be set in the header of the executable image. Ideal for 32-bit database apps on 32-bit architecture.  
32-bit apps on 64-bit architecture can take advantage of a mechanism called **_Address Windowing Extensions_** (_AWE_) which allows a 32-bit app to allocate up to 64 GB physical memory and then map "views" (or _windows_) of the physical memory onto the app's 2 GB virtual address space. Windows does not automatically take advantage of Address Windowing Extensions. The app developer must code AWE-support into the 32-bit app. 

64-bit Windows 8 provides 8 TB/8 TB virtual memory for user process / system address space. 64-bit Windows 8.1 provides 128 TB/128 TB virtual memory for user process / system address space. The middle space is unmapped. Yes, it theoretically could be used but isn't.   
So in Windows 8.1 the user process address space is 0x0000000000000000 through 0x7FFFFFFFFFFFFFFF. 128 TB shakes out to 2^48 bits / 48-bit space. 4 GB -> 2^32 bits / 32-bit space. Virtual memory address space for system processes is 0xFFFF000000000000 through 0xFFFFFFFFFFFFFFFF. 

64 bits of address space corresponds to 16 exabytes. However, real-world 64-bit hardware will probably never support 16 EB of virtual memory. 

## User Mode vs. Kernel Mode
Let's descend to a layer of abstraction beneath the OS for a moment. Intel & AMD processors feature hardware-enforced hierarchical protection domains called _protection rings_. Ring 0 is most privileged and Ring 3 is least. The Windows operating system uses only Ring 0 & Ring 3. Ring 1 and Ring 2 are not used.

Windows calls these processor access modes _kernel mode_ for Ring 0 and _user mode_ for Ring 3. "Kernel mode" refers to a mode of code execution that allows access to all system memory & CPU instructions. The motivation underpinning the implementation of access modes is to prevent user apps from reading/writing critical OS data.

User application code runs in user mode while OS code like system services & device drivers runs in kernel mode. By providing the OS kernel a higher privilege level than user apps, CPUs provide OS designers a necessary foundation for ensuring that malware can't disrupt the stability of a whole system. 

The system process (aka kernel-mode OS?) and device driver code share a single virtual address space. This is the exception to the rule that all processes are assigned their own private virtual address space. The aforementioned virtual address space is also included in the address space of every other user process. The OS tags each 4 KB page of virtual memory with the access mode the CPU must be in to read or write that page. The virtual memory address pages associated with system space [0xFFFF000000000000 -> 0xFFFFFFFFFFFFFFFF] can be accessed only from _kernel mode_ while all pages in the user process address space [0x0000000000000000 -> 0x7FFFFFFFFFFFFFFF] are accessible from both _kernel mode_ & _user mode_. Read-only pages (those that contain static data and therefore will never, ever change) are not writable from any mode. 

A minimum requirement to install Windows is that the CPU support NX (No-Execute) and DEP (Data Execution Prevention.)  
> No-Execute is the generic industry term. AMD calls it Enhanced Virus Protection while Microsoft calls it Data Execution Prevention.
Windows will flag pages of virtual memory that just contain data as 'non-executable,' thereby preventing malware from injecting executable code into those pages. For example, Windows probably flags the set of virtual memory pages associated with a .pk3 file sitting in physical memory as non-executable. It goes without saying that DEP should never be disabled. (Maybe I needed to disable NX/DEP in the BIOS before running Mimikatz?)  

Once a component (whatever that means) is running in kernel mode there is _no protection_ against read/write operations targeted at the system virtual address space. Stated differently, OS and device driver code that's being executed in kernel mode can perform any operation on the system space of virtual memory. All features of Windows security are bypassed. And because the majority of code that comprises the Windows operating system runs in kernel mode, Microsoft developers rigorously design and test system-level components to ensure they don't violate system security or cause system instability. 

Now you know why third-party device drivers should always reflect as digitally signed when the UAC window appears. Device drivers are a perfect attack vector for malware to gain full read/write access to all OS system data. 

> DETOUR: 64-bit & ARM version of Windows 8.1 feature the kernel-mode code-signing (KMCS) policy that mandates all device drivers be signed with a cryptographic key assigned by one of the major code certification authorities. Even if the user has local admin they are unable to force the installation of a device driver not signed by the list of major code certification authorities. However in the pre-boot environment of F8 under Advanced Startup Settings you can semi-disable the kernel-mode code-signing policy. The KMCS policy is re-enabled at the next reboot so disabling the KMCS policy does not persist across reboots. This so-called Test Mode is for developers.  
> -- Windows 10 locked down the requirements for signing device drivers even further. All new Windows 10 drivers must be signed by two (?) of the [accepted code certification authorities](https://docs.microsoft.com/en-us/windows-hardware/drivers/dashboard/get-a-code-signing-certificate#step-2-buy-a-new-code-signing-certificate). The driver must be signed with an Extended Validation (EV) Hardware certificate that's encrypted with SHA-2. Regular file-based SHA-1 certificates are no longer acceptable. Once the hardware device driver is EV-signed the developer submits to the Partner Center for attestation signing. Microsoft will then sign the driver themselves. This means that Windows will only install device drivers signed by Microsoft, with no exceptions aside from Test Mode.  
> -- Windows Server 2016 is even more selective. In addition to being signed with an approved EV cert & attestation from Microsoft, a proposed device driver must pass through strict Windows Hardware Quality Labs (WHQL) certification as part of the Hardware Compatibility Kit (HCK.) On Server 2016 & beyond, only WHQL-signed drivers, which provide certain compatibility, security, performance, and stability assurances, are allowed to be installed.  
> -- Certain vendors, platforms, and even enterprise configurations of Windows may request a customization of these driver-signing policies, like what's available through **Device Guard**. For example, a customer may request that their Windows 10 hosts only install device drivers signed by WHQL... or going the opposite direction it's entirely possible for an enterprise customer to request that their instances of Server 2016 not face the WHQL-signing requirement. 

Threads of user-mode processes switch from user mode to kernel mode when they make a _system service call_. For example, a call into the Windows _ReadFile_ API eventualy needs to call the internal Windows routine that actually handles reading data from a file. That routine, because it accesses internal system data structures, must run in kernel mode. The transition of that internal Windows routine from user mode -> kernel mode is accomplished by running a special processor instruction that causes the processor to enter the system service dispatching code into the kernel. This in turn calls the appropriate internal function in **ntoskrnl.exe** or **Win32k.sys** so in the case of the _ReadFile_ API, the appropriate internal function is the _NtReadFile_ kernel service function. Kernel service functions validate parameters and perform appropriate access checks using the Security Reference Monitor before they execute the requested operation. When the function finishes and control must be returned to the user process thread, the OS switches the processor mode back to user mode to prevent read/write operations of critical OS system data by other processes. **CHAPTER 2 OF THE WINDOWS INTERNALS BOOK CONTAINS THE FULL EXPLANATION**. 

So a... thing... runs in the context of either 1.) _kernel mode_ or 2.) _user mode_. Processes, device drivers, the Windows Executive, and the kernel can all run in one of two "access modes:" kernel mode or user mode.     
All processes, **except for the System process**, run in user mode. Device drivers and OS components like the Windows Executive (more on that later) and the kernel run only in kernel mode.

>Microsoft defines the Windows Executive as “kernel mode components that provide a variety of services to device drivers, including object management, memory management, process and thread management, input/output management, and configuration management.” 

## Objects
[Windows defines 3 categories of object types](https://docs.microsoft.com/en-us/windows/win32/sysinfo/object-categories). Realize here we're talking about the _categories_ of object types. They are:
1. The **user** category of object types. This collection of objects supports window management. 
2. The **graphics device interface (GDI)** category of object types supports graphics 
3. The **kernel** category of object type reflects object types for memory management, process execution, and interprocess communications (IPC)

In Windows a **_kernel object_** is a single run-time instance of a statically defined object type. An **_object type_** is comprised of:  
1. A system-defined data type  
2. Functions that operate on instances of the data type and   
3. a set of object attributes.  

The Windows app developer will encounter process, thread, file, and event objects, for example. These objects are based on lower-level objects that Windows creates and manages. Other kernel object types include communication ports and shared memory sections. Are mutexes, events, and semaphores kernel objects or synchronization objects?  

In Windows, a _process_ is an instance of a _process object type_. A _file_ is an instance of a _file object type_. You get the idea. An _object attribute_ is a field of data in an object that partially defines the object's state. So a process object will have object attributes like the PID, a base scheduling priority, and a pointer to another object of type _access token_. 

An _object method_ is a tool for performing read/write operations on an object. More specifically, object methods are a means for performing R/W on _object attributes_. For example, the **_open_** method for a process would accept a PID as input and return a pointer to the process object as output. 

The primary difference between an object and any other data structure is that the internal contents of an object are only visible/writable by calling an object service. It's not possible to "open" an object to view/modify the contents within like other data structures. This difference separates the underlying implementation of an object from the code that merely uses it, and allows object implementations to be changed easily over time (?)

A kernel component called the _Object Manager_ provides objects with a convenient means for accomplishing the following four important OS tasks:  
1. Providing human-readable names for system resources
2. Sharing of resources and data among processes
3. Protection of resources from unauthorized access and 
4. Reference tracking, which allows the system to recognize when an object is no longer in use so that it can be automatically deallocated. 

Not all data structures in Windows OS are objects. Only data that needs to be shared, protected, names, or made visible to user-mode programs (via system services) is placed in objects. Data structures used by only one component of the OS to implement internal functions are not objects. **CHAPTER 8, PART 2 OF WINDOWS INTERNALS DIVES INTO OBJECTS AND HANDLES**

>**WinObj** is the Sysinternals utility for viewing object instances

## Handles (a reference to an object instance)
The kernel-mode core of Windows consists of various subsystems like the Virtual Memory Manager, Process Manager, I/O Manager, Object Manager, Security Reference Monitor, IPC Manager, PnP Manager, Power Manager, and Configuration Manager (registry). The executable that kicks off the kernel-mode core of Windows is **ntoskrnl.exe**. The aforementioned components are subsystems of the Windows Executive. Each of these subsystems defines 1 or more object types with the Object Manager to represent the resources they expose to apps. For example:    
* Configuration Manager defines the _Key_ object type to the Object Manager. The Key object type represents an open registry key. 
* Virtual Memory Manager defines the _Section_ object type to represent shared memory (?) 
* I/O Manager defines the _File_ object to represent open instances of device-driver resources, which include file-system files.
* Process Manager defines the object types of _Thread_ and _Process_.  
* The Windows Executive defines object types of _Semaphore_, _Mutant_ (which is the internal name for a mutex, used for mutual exclusion), and _Event_. Semaphore, Mutant, and Event are examples of object types for "synchronization" or something. (these 'synchronization objects' are objects that wrap fundamental data structures defined by the operating system's Kernel subsystem.)

The above is not even close to a complete list of object types in the Windows operating system. Run the **WinObj** utility of the Windows Sysinternals tools as Admin to see the current list. Windows 10 defines appx 53 object types. 
 
When an app wants to use a resource of type _Key_, _File_, _Mutant_, etc., the app must first call the appropriate API function to create or open the resource. _CreateFile_ opens or creates a file. _RegOpenKeyEx_ opens a registry key. _CreateSemaphoreEx_ opens or creates a semaphore. If the function suceeds in creating/opening the resource then Windows allocates a reference to that object instance in the _**handle table**_ of the process and returns the index (handle value) of the newly created entry in the handle table to the app. This handle value is how the app pinpoints the resource when subsequent operations are performed on that resource. So to query (read) or manipulate (write) a resource the app passes the appropriate handle value to the API function that lines up with the desired operation. The system finds the corresponding entry in the handle table and produces a pointer to the resource / object instance and from there the API function performs its operation. 

Handles in a handle table also store the privileges a process was granted when it created / opened the resource. Uhhh apparently when a process creates / opens an object instance it also declares which operations it intends to perform on that resource... 

Let's get this straight: the application wants to perform the _WhateverAPI_ function on a particular resource... The system finds the row in the handle table that corresponds to the resource's handle value... and then the system checks the privileges the process was issued when opening/creating the resource by referring to yet another field in the handle table... if the process passes this Authorization of sorts then the system will find the resource referenced by the contents of the pointer field and apply the _WhateverAPI_ function to that resource. 

For example, if a process opened a file just to perform a read operation but later references that same handle value to perform a write operation then that API function would fail to write.

A process that no longer needs to access an object instance can release its handle to that object by passing that handle value to the _CloseHandle_ API function. When a process exits, any handles in the handle table are closed.  

>The Windows Executive manages each process' handle table. 

>>REVIEW the section in Application Isolation titled **Protected Processes**. That's another process type that would fit well. 