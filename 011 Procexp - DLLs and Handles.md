# DLLs & Handles
DLL view lists all the dynamic-link libraries and other files mapped into the process' address space. Handle view lists all the kernel objects opened by the process. 

## Finding DLLs and handles
One of Process Explorer's most powerful features is to identify the process that has a DLL or kernel object open.  Procexp is useful for when a file cannot be opened / saved / closed / deleted because 'The file is open in another program' but the dialog box doesn't give any further details. 

Hit **CTRL+F** to Open Process Explorer search and input a string or substring of a DLL name, handle type, or handle name to find the PIDs that match. 

DLL view displays all DLLs and memory-mapped files like data files and the image file (so the .exe) being run. For the System process DLL view lists all image files mapped to kernel memory like **ntoskrnl.exe** & all the loaded device drivers. DLL view is empty for the System Idle Process and Interrupts because they technically aren't processes. Same for protected processes. You're not allowed to view the DLL files opened by **smss.exe**

Let's go through the nontrivial columns in the DLL view:

**Image Base** For files loaded as executable images, this column holds the virtual memory address from the header of the executable image file. This virtual memory address is where the image _should_ be loaded. If any of the necessary memory range is already in use, the image needs to be relocated to another address.     

**Base** Virtual memory address where the file is _actually_ located.   

**Size** the number of contiguous bytes, starting from the base address, consumed by the file mapping. The size is represented as a hexadecimal number. Do not forget that _BYTES_ are being represented. So a Size of 0xFF is 255 bytes.    

**Mapping Type** Displays _Image_ for executable image files and _Data_ for data files. DLL files listed as _Data_ are loaded for resources only (like such as icons or localized text) and unnamed file mappings.  

Under the colors management dialog box select Relocated DLLs. These are memory-mapped files (remember that DLL view contains more than just DLL files) where the memory manager couldn't allow for the memory-mapped file to be loaded into the Image Base address (because it was already occupied) and had to be relocated by the loader. This consumes CPU and makes parts of the memory-mapped file that are modified as part of the relocation not sharable. This reduces the efficiency of Windows memory management. _Windows Internals_ contains 180 pages of dense material on memory management. 

## Peering deeper into DLLs
Open the Properties dialog box of a random memory-mapped file and inspect the [Image tab](https://i.imgur.com/JBpaTGf.png). What's cool is that hitting Explore opens File Explorer with that object selected. Hit the Verify button to confirm that the file is signed by an authority whose chain of trust connects up to one of the certs in Windows. 

Now select the Strings tab. In computer programming a "string" is a data structure consisting of a sequence of characters, usually representing human-readable text. Process Explorer inspects the code of that memory-mapped file and displays all strings of length 3 or more. Flip between Image and Memory radio buttons to view the results of that memory-mapped file read from disk or system memory, respectively. It is possible for the file image on disk to differ from the corresponding memory-mapped file in RAM.   

## Handle view
Displays the object handles belonging to the process.  
>NOTE: **Object handles** are what programs use to manipulate system objects managed by kernel-mode code. Such system objects include, but are not limited to files (File), a communications endpoint (File), a driver interface (File), registry keys (Key), synchronization objects (Semaphore, Mutant, Event), partitions (Partition), processes & threads, symbolic links (SymbolicLink), memory sections (Section), FilterConnectionPorts (?), TS sessions (Session), TS window stations (WindowStation), security tokens (Token), TS desktops (Desktop), and APLC ports 

When a process tries to create or open an object, it also requests specific access rights (from what?) for the operations it intends to perform, such as read and write. If the create or open action is successful, the process acquires a handle to the object that includes the access rights that were granted. That handle will then be used for subsequent operations on the object, but only for operations where the access right was granted. The access rights of the logged-in user account are irrelevant. The local user could have Full Control over the object but the process must have the appropriate access rights to perform the corresponding operation on the object. 

At the app level a handle is merely an integer. That integer serves as a byte offset into the process' handle table, which is managed in kernel memory. Information contained in the rows of the handle table include the object type, the access granted, and a pointer to the data structure representing the actual object. 

>CONFUSION: Loading a memory-mapped file into the process' address space does not normally add a handle to the handle table. Such files can therefore be in use and not be able to be deleted, even though a handle search might come up empty. 

**Type** the object type of the securable object for which the process received access. Object types include Directory, Semaphore, Desktop, File, Key, Thread, etc.   

**Name** The name associated with the object. For most object types the name is an _object namespace name_ like `\Sessions\2\Windows\WindowStations\WinSta0` or `\Device\Nsi`.  
Format of Name for process handles is just the .exe file with the PID like **chrome.exe(12812)**     
Format of Name for thread handles is just the Name of the process handle with the thread ID concatenated **chrome.exe(12812): 23726**.    
Format of Name for token handles is the security principal and the logon session ID: **NT AUTHORITY\SYSTEM:3e7**, **NT AUTHORITY\NETWORK SERVICE:3e4**, **NT AUTHORITY\LOCAL SERVICE:3e5**, and **RYZEN3900X\CarlS:c48cb** are some examples. 

>To the naked eye, the Name of a file system object or registry object looks like the familiar directories & registry keys you're familiar with. However, this is just an abstraction. The Windows operating system internally understands file & registry objects differently.  
>>A file system object's drive letter is replaced with something else, as is the registry's root key. Specifically, the Name of a file system object replaces the drive letter. For example, the Name of a file object that appears as `C:\SysInt\Procexp.exe` is internally `\Device\HarddiskVolume1\SysInt\Procexp.exe`.  
>
>>The Name of a key object has its root name replaced. The following is a pairing of friendly name of a root key and the proper name of that root key known to the Windows kernel:.      
>>`HKCR` -> `\REGISTRY\MACHINE\Software\Classes\`  
>>`HKCU` -> `\REGISTRY\USER\<user_sid>\`    
>>`HKLM` -> `\REGISTRY\MACHINE\`  
>>`HKU` -> `\REGISTRY\USER\`  
>>So if the Name of a key object is **HKLM\SOFTWARE\Microsoft\Ole** the Windows kernel knows that name is actually `\REGISTRY\MACHINE\SOFTWARE\Microsoft\Ole`   
>> `HKCC` holds no data and is just a pointer to `HKEY_CURRENT_CONFIG\SYSTEM\CurrentControlSet\Hardware Profiles\Current`

**Handle** A hexadecimal number that the process passes to an API to access the corresponding object instance and perform an operation. This hexadecimal number is the byte offset into the process' handle table as it's represented in virtual memory.  

**Access** The bitmask in hexadecimal that identifies what permissions the process is granted through the handle. Each bit that is set grants a permission specific to the object type. This bitmask is properly known as an _access mask_. [Access mask format](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-mask-format). 

**File Share Flags** Only pertains to file objects. This is the _sharing mode_ (?) that was set when the handle was opened. Flags can include R (read), W (write), and/or D (delete)... And only in that order. This indicated that other callers (including calling threads within the process) may open that same object and perform those RWD ops. The absence of a permission is marked with a dash. For example, a process that has read and delete but not write on a file object will have the file share flag of **R-D**.  
>A file object whose file share flag is blank means that object is open for use exclusively through this handle (?) I don't get this. The file system object `\Device\Nsi` has no flags yet multiple processes can access. 

**Object Address** The memory address in kernel memory of the data structure representing the object. This address is useful when using a kernel debugger.  

**Decoded Access Mask** The translated contents of the **Access** column to human-readable text. 

Navigate to the menu and select File -> Show Unnammed Handles and Mappings. Many more entries appear in the DLLs & Handles view. These are typically object instances that the process creates for its own use and possibly for the use of a child process, so long as the child had a way to identify which inherited handle value it should pass to an API. 

Handles can also be duplicated from one process to another, provided the process performing the handle duplication has the necessary access right to the target process. 

Right-click a handle an a context-sensitive menu of 2 entries arises. Telling you right now. It's risky to hit Close Handle. Could cause your machine to BSoD. Could corrupt data!!! Could do nothing. Do not manually close a handle with Process Explorer unless you know what you are doing.   

>CONFUSION: Then the book just tosses in your lab something called a 'reference' which I can not, for the life of me, make heads or tails of from the book alone. Like, seriously, WTF is this (?)   
>>   ![asdf](https://i.imgur.com/XPq4wHi.png)   
> How is 65456 of anything related to that test .docx file?!? I'm going to just put this to the side and move on.   