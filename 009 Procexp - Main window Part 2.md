# Main window - column selections to display process attributes
In the Handles table, DLLs table, and process list you can right-click Select Columns and view a wealth of information. 100+ columns available for processes and 40+ for Handle & DLL tables. Let's understand the contents of each of these tabs. 
## Process Image 
**User Name** User account of the process in DOMAIN\USER format.  

**Description** Extracted from the version resource (?) of the executable image. I think they meant File Description? Not important.  

**Company Name** Extracted from the version resource of the executable image. If Company Name column isn't present that value appears at the top of the tooltip box.  

**Verified Signer** Confirms whether the executable image is verified as digitally signed by a certificate that chains to a root authority trusted by the computer. Uhh this column is completely blank for all running processes.  

**Version** file version extracted from the version resource of the executable image.  

**Image Path** Path to executable image. Like Company Name, if enabled then the directory doesn't appear on tooltip.  

**Image Type (64 vs 32-bit)** Is the process executing 32-bit code in 32-bit compatibility mode via the Windows on Windows64 (WoW64) subsystem? Of is the process executing native 64-bit code?  

**Package Name** Only pertains to UWP apps. It's just the result of  
```PowerShell
(Get-AppxPackage).PackageFullName
```

**DPI Awareness** On 8.1/2012R2 and newer. Reports the process' level of [DPI awareness](https://docs.microsoft.com/en-us/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows#dpi-awareness-mode). The three possible values are _Unaware_, _System Aware_, and _Per-Monitor Aware_. This is related to how an app's UI renders & scales upon a display's DPI update.  

**Protection** Is the selected process a protected process, and which type of protected process?  

**Control Flow Guard** Was the process' image file built with Microsoft Visual Studio's Control Flow Guard protection? Values are either CFG, n/a, or (blank)    

**Autostart Location** Indicates the Registry location where the process is configured to start automatically, is a location has been specified. See [this example](https://i.imgur.com/9yiXyeV.png).  

**DEP Status** Is Data Execution Prevention enabled for the highlighted process? DEP is a security feature that mitigates buffer overflow and other attacks by disallowing code execution from memory that has been marked "no-execute," such as the _stack_ and _heap_ (?)    
>CONFUSION: I was always under the assumption that DEP is a hardware-level concept for CPUs, and that a CPU isn't aware of what a "process" is. Turns out I was quite confused. DEP is (a.) enabled in hardware (b.) enabled by default in Windows to varying degrees of enforcement strength, and (c.) enabled in the executable image. System-default status of Data Execution Prevention [here](https://i.imgur.com/9pC1KuG.png).  

**Integrity Level** You already understand System, High, Medium, and Low. UWP apps by default run at **Low** IL but are reported as "AppContainer" becuase they are subject to additional restrictions. Probably not a bad idea to review Application Isolation.  
>**chrome.exe** creates child instances of chrome.exe that run at an integrity level of **Untrusted**.  

**Virtualized** What Process Explorer means by "virtualized" isn't obvious. Indicates whether <span style="color:#46FF46;font-size:1em;">**UAC file and registry virtualization**</span> is enabled for the running process. File and registry virtualization is an application-compatibility technology that intercepts attempts by legacy (i.e. 32-bit) Medium IL processes to write to protected areas, and transparently redirects these attempts to areas owned by the user.  

The concept of file & registry virtualization does not apply to 64-bit applications! Run the 32-bit application doomseeker.exe in a non-elevated security context. [Process Explorer reflects the Medium IL process as _virtualized_](https://i.imgur.com/FG4kwPV.png). It's important to realize, however, that running a 32-bit app in an elevated security context defeats the whole reason that file & registry virtualization exists in the first place because since the process runs at the High IL the protection doesn't apply. [Don't do this.](https://i.imgur.com/bfzwobO.png)   

>ASIDE: [File & Registry Virtualization in Windows 10](https://www.thewindowsclub.com/file-registry-virtualization-in-windows-7) has nothing to do with hardware virtualization or OS virtualization. It's just overly complicated language for stating that Windows prohibits certain varieties of processes from writing to certain system files & registry keys.   
When file & registry virtualization is enabled any legacy 32-bit application that attempts write operations on `HKEY_LOCAL_MACHINE\Software` will have that operation redirected to `HKEY_CLASSES_ROOT\VirtualStore\Machine\Software`.  
>> CONFUSED: Why does Process Explorer provide a column for whether a process has file & registry virtualization enabled when it's actually a system-wide setting? Is it possible to disable file & registry virtualization on a per-executable image basis?   
>  
> [Microsoft documentation on Registry Virtualization](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry-virtualization)  

**ASLR Enabled** Any application worth a damn will have _Address Space Layout Randomization_ enabled in the executable image file's header. ASLR is a defense-in-depth security feature that can mitigate remote attacks that assume function entry point addresses (remember what we learned in the chapter on call stacks) in virtual memory are predictable. 

>ASIDE: Process Explorer reflects the _dynamic ASLR state_ of doomseeker.exe under Properties -> Image as "Enabled (permanent)Disabled" while zandronum.exe indicates the _dynamic ASLR state_ as "Bottom-Up." Another item of interest is that the ASLR entry for doomseeker.exe in the process list is simply blank while that column for zandronum.exe reads ASLR. *winword.exe* reads "High-Entropy, Bottom-Up, Force-Relocate"  

**UI Access** Not all processes with a thread that own an instance of a window object are automatically subject to User Interface Privilege Isolation. UI Access (i.e., UIPI bypass) is intended mainly for accessibility software. However, as demonstrated [here](https://i.imgur.com/6I0XNua.png), UI Access was given to Radeon Settings so hopefully Microsoft has in place a security mechanism for protection against shatter attacks when a process is not UIPI-enabled.   

**Enterprise Context** This is related to [Windows Information Protection (WIP)](https://docs.microsoft.com/en-us/windows/security/information-protection/windows-information-protection/wip-app-enterprise-context) which is a subject matter unto itself. "Enterprise Context" is only applicable in a domain environment. The three values are _Domain_, _Personal_, and _Exempt_. This isn't directly related to the Sysinternals utilities so review the link later.

## Process Performance 
**Tree CPU Usage** displays what's essentially a Subtotal of CPU consumption by a process and its descendants. Accounting is time-based like Task Manager and not calculated from CPU cycles.  

**CPU History** a graphical representation for each running process crammed into one cell. Kernel mode time is in red and user mode time is in green. Running Cinebench or Doom Eternal only produced green.  

**CPU Time** The total duration of kernel mode + user mode time for the highlighted process. This column does **_NOT_** display cumulative CPU time for a process and its descendants.  

**Process Timeline** Graphical representation of when a process started relative to system start time.  

**Base Priority** The column header in the process list reads "Priority" so try to keep that in mind. 8 means Normal. 13 means High. So in the Windows operating system the concept of a Priority value follows that a lower-valued number equate to a lower priority. Contract with Spanning Tree Protocol where a high priority means low number. Stating that a process in Windows is "Priority #1" means it's unimportant with regards to resource allocation.  

**Handle Count** The number of handles to kernel objects currently open by the running process.  

**Threads** self-explanatory.   

**CPU Cycles** & **CPU Cycles Delta** Former is total sum of kernel mode + user mode CPU cycles consumed by the process instance since launching and latter is change since Process Explorer last updated the on-screen data.     

**Context Switches** & **Context Switch Delta** Incremented when the CPU context changes to begin executing a thread in a process instance whose status is Running. Start a WUP app and minimize the window. Context Switches value halts and CSwitch Delta drops to 0.  
>ASIDE: The values in the Context Switches and CSwitch Detla columns for the Interrupts pseudo-process represent the quantity of CPU interrupts (aka hardware interrupts) and DPCs.  

## Process Memory
Obviously these columns reveal memory metrics but they also hold counts to the windowing system's GUI and USER objects. Review what the _Working Set_ and _Private bytes_ are! Refer to [this article on physical and virtual memory in Windows 10](https://answers.microsoft.com/en-us/windows/forum/windows_10-performance/physical-and-virtual-memory-in-windows-10/e36fb5bc-9ac8-49af-951c-e7d39b979938).   

**Page Faults** & **PF Delta** The total (or change in) number of times that the process instance accessed an invalid memory page, causing the fault handler of the Virtual Memory Manager to be invoked. This happens surprisingly frequently. Reasons for pages being "invalid" are:  
1. The page is on disk in a paging file or mapped file.  
2. Accessing that page for the first time requires copying or "zeroing" (whatever that means) 
3. There was illegal access resulting in an access violation. Like a process of low IL trying to modify a page belonging to a core Windows process at System IL?  

These columns also lump in the total number of "soft page faults" where the Virtual Memory Manager was able to resolve the fault by referencing info not in the working set of the process instance but already somewhere else in physical memory. 

**Minimum Working Set**: Amount of physical memory reserved for the process. Windows guarantees that the process' working set receives at least this number of pages in RAM. Remember that the default size of a page is 4KB. Now take an example where the minimum working set for a process is _N_ pages. That process can lock pages in the working set up to N-8 pages. (?) Note that a _minimum working set_ does not guarantee that a process' working set will always and in all cases be at least that large. After all, if a process isn't executing or consuming resources then it doesn't need much physical memory anyway.  

**Maximum Working Set**: Max amount of physical memory assigned to the working set of a process. Windows ignores this number unless a hard limit is in place by a resource-management application. 

**Working Set Size** & **Peak Working Set Size**: The amount of physical memory assigned to the process by the Virtual Memory Manager subsystem. Peak working set size contains the instance's highest measured value of Working Set Size since launch.   


**Private Bytes**, **Private Delta Bytes**, **Peak Private Bytes**, and **Private Bytes History** Private bytes refers to the quantity of physical memory allocated & committed for tasks dedicated to only that process. As such, private bytes for Process A are not sharable with Process B.  
>CONFUSION: For way too long I thought that whether memory was part of "private bytes" or the "working set" depended on which entity allocated those bytes but I'm starting to suspect that's not relevant.    

Executing **winword.exe** loads that image from disk to memory and creates an instance of the winword.exe process. The data in memory that comprises winword.exe sits in the sharable bytes of the Working Set. This is because its conceivable that additional process instances of winword.exe could be launched at the same time to open more than one document. To access the winword.exe entity in RAM those other PIDs could also refer to the same memory address space.  
Double-clicking multiple different .docx files in File Explorer loads those files into the private bytes of the corresponding instance's working set. There is no need for the winword.exe instance of one Word document to directly access the data comprising another .docx file open in another instance. 

**WS Sharable Bytes**: The quantity of a process' working set that could conceivable, possibly, in theory be shared with other processes. An example would be a mapped executable image like excel.exe. 

**WS Shared Bytes**: The portion of a working set's memory that is currently shared with other processes. 

**WS Private Bytes**: "The portion of the process' working set that contains private bytes that cannot be shared with other processes."

>How the fuck does that ^ differ from the Private bytes column?! (?)

**Virtual Size**: Review the Virtual Memory section in your markdown file for _Processes, Threads, and Jobs_. Virtual Size column signifies the amount of virtual memory that has been reserved or committed for the process.  
> The difference between reserved virtual memory and committed virtual memory will be explained in time.   

64-bit processes with Control Flow Guard (CFG) support are always reserved a virtual size of 2 TB!  
>CONFUSION: Uhhh then why can we have more than 64 instances of processes that support 64-bit user apps? What am I missing here? Do these "reserved or committed" ranges of virtual memory overlap? (?) 

Control Flow Guard reserves a 2 TB region to support its bitmap of valid indirect-call targets in the 128TB virtual memory address space dedicated to user processes. I understood probably half of that sentence. (?) x86 processes reserve up to a 64 MB region. Then why the hell is the virtual size of doomseeker.exe 200 MB (?)  
>NOTE: Do **_NOT_** confuse yourself by thinking that the "virtual size" of a process is the number of bytes in virtual memory that signify an instance's quantity of physical private bytes + physical shared bytes. Because if you ponder that for a few moments, you realize it doesn't make any sense.  

**Memory Priority** column contains the default memory priority that's assigned to the instance's pages of physical memory. Pages that are cached in RAM and not part of the working set of any process get repurposed, starting with the lowest priority. On my Ryzen3900X machine the Memory Priority for all processes is 5, with the lone exception being **SettingSyncHost.exe** which is 1.   

**GDI Objects** According to blogs.solidworks.com "GDI Objects (Graphics Device Interface) is a core windows component responsible for representing graphical objects and outputting them to devices such as printers or monitors. For every window or application that is open, it uses up GDI Objects." GDI objects include brushes, fonts, and bitmaps. 

**USER objects** USER objects are entities like windows and menus. 

>NOTE: GDI & USER objects are created by the windowing subsystem in the process' terminal server _session_. GDI objects & USER objects are not kernel object types like those found in the Handles pane of Process Explorer, so there is no security descriptor associated with this objects. 

**Paged Pool** & **Nonpaged Pool**: the amount of paged/nonpaged pool charged to the process. 

### Further reading
[Pushing the Limits of Windows: Virtual Memory](http://blogs.technet.com/markrussinovich/archive/2008/11/17/3155406.aspx)





```azurepowershell
GET-PROCESS dafsdafasd sdfsadfas asdfsadf sadfsadfas sdafsad sdafsadf sadfsadfsad sdafafsadf sadfsadf 
RESTART-VM
```


> [!TIP]
> how about an alert?
> 