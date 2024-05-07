# Call Stacks and Symbols
Process Explorer, Process Monitor, and VMMap allow us to track the paths when a particular piece of code is executing at a particular time. This path is known as a _call stack_. Associating _symbol files_ with _modules_ in a process' address space provides more meaningful context information about its code paths. Understanding call stacks & symbols, and how to configure them in Sysinternals utilities, gives deep insight into a process' behavior and can often lead to the root cause of many troubleshooting problems. 

## Call Stacks
Executable code in a process is normally organized as a collection of discrete functions. A function can invoke other subfunctions to perform its tasks. When a subfunction as finished its task it returns control back to the calling function. 

The book's example of a function-calling sequence: MyApp.exe ships with a DLL file **HelperFunctions.dll**. Inside that DLL file is a function named _EncryptThisText_ which encrypts text passed to it. _EncryptThisText_ consists of _n_ instructions. Some instructions are handled by _EncryptThisText_ itself. Other instructions call other functions. Let's assume _EncryptThisText_ runs a few of its own instructions and then comes to an instruction that calls the function _CryptEncryptMessage_. After performing some preparatory operations (?) _EncryptThisText_ calls the Windows API (Win32) function _CryptEncryptMessage_. The _CryptEncryptMessage_ function is stored in **Crypt32.dll**. At some point the _CryptEncryptMessage_ function will need to have memory allocated to perform operations so it will call the _malloc_ function that's stored inside **Msvcrt.dll**. 

Assuming that _malloc_ doesn't need to call a function, control is returned to _CryptEncryptMessage_ once _malloc_ has completed its memory allocation task. Execution resumes where _CryptEncryptMessage_ left off. Are you seeing a pattern here? Once the Win32 API function _CryptEncryptMessage_ finishes the job control is returned to _EncryptThisText_. Not too hard to grasp. 

A **_call stack_** is the construct that enables the system to know _how_ to return control to a series of callers, as well as how to how to pass parameters between functions and to store local function variables. Call stacks are organized in a LIFO manner, where functions remove items in the reverse order from how they add them (what??) Let's review a simple example:  
>MyApp.exe executes Function A. Function A runs through some instructions and then comes to an instruction that calls Subfunction A. Before calling (or performing a function internal to Function A for that matter) Function A writes the memory address of the next instruction to execute after Subfunction A has completed (the _return address_) at the top of the LIFO queue. Last In First Out (LIFO) is the order in which you remove weights from a bench press bar. The book calls this LIFO queue the stack. When Subfunction A calls another function, Subfunction A adds its own _return address_ to the top of the stack. On returning from a function, the system retrieves whatever address is at the top of the stack and resumes executing code from that point. 

The convention for describing the return addresses in a call stack is

<span style="background-color:Black;font-family:Consolas;font-size:1.2em;border:2px Solid MediumSeaGreen;">_module_!_function_+_offset_</span>. _module_ is the name of the executable image file (.exe/.dll) that contains _function_. And _offset_ is the number of bytes in hexadecimal past the beginning of _function_... I don't know what that means. (?) For function names that are unavailable the format is _module_+_offset_. 

While the _malloc_ function is running the call stack for the path from _MyApp.exe_ to _malloc_ could look something like this:
```
msvcrt!malloc+0x2a
crypt32!CryptEncryptMessage!0x9f
HelperFunctions!EncryptThisText!0x43
MyApp.exe+0x25d8
```
Understand that call stacks are a constantly changing data structure and allow us to determine the call sequence & memory allocation when a piece of code is executing. 

## Symbols
(?) When a debugger inspects a thread start address or a return address on a call stack it can uncover what module the executing code belongs to by examining the list of loaded modules & their address ranges. However, when a compiler converts a developer's source code into computer instructions, the compiler does not retain the function names. Usually. The one exception is taht a DLL file includes an _export table_ that lists the names & offsets of the functions it makes available to other modules. However, the DLL file's export table does **_NOT_** list the names of the library's internal functions, nor does the export table list the names of COM entry points that are designed to be discovered at runtime (?)

>NOTE: Executable files loaded in user-mode processes are generally either a. EXE files with which a new process can be started or b. DLL files that are loaded into an existing process.  
> EXE and DLL files are not restricted to using the .dll and .exe file extensions. Files with COM and SCR extensions are actually EXE files... while ACM, AX, CPL, DRV, and OCX are examples DLL files that aren't of filetype .dll.  
>> so **"C:\Windows\System32\powercfg.cpl"** is actually a DLL file?
>
>Installation programs commonly extract and launch EXE files with TMP extensions.

When creating executable files, compilers and linkers can also create corresponding _symbol files_ (or _symbols_) with default extension of PDB (short for program database.) Can you conceptualize symbols as analogous to log files? They hold data that is not needed when running executable code, but which can be useful during debugging. This info includes the names and entry-point offsets of functions within the module (usually a .dll file) Using this info, a debugger can take a memory address and find the function with the closest preceding address. 

In the absence of symbols that the compiler creates a debugger doesn't have much to go on. Windbg would only have exported functions to work with, and those might not have any relation to the code being executed. In general, the larger _offset_ of a return address, the less likely the reported function name is to be accurate (?)

>NOTE: The Sysinternals tools are only able to use native (aka unmanaged) symbol files when reporting call stacks, and are not able to report function names within JIT-compiled .NET assemblies (?)

Systems must build the symbol file at the same time as running its corresponding executable. Otherwise it will be incorrect and the debug engine might refuse to accept it for analysis.  
> Older versions of Microsoft Visual C++ created symbol files only for Debug builds unless the developer explicitly changed the build configuration. Newer versions now create the symbol files for Release builds as well, writing them into the same directory as the executable files. Microsoft Visual Basic 6 could create symbol files, but not by default. 

Developers rarely allow symbol files to contain full detail because that's a pathway to successfully reverse-engineering the company's proprietary software. They keep these _full symbol files_ (sometimes called _private symbol files_) to themselves. Full symbol files contain info like (a) the path to and the line number within the source file where the symbol file is defined, (b) function parameter names and types (c) variable names and types. Companies usually only allow _public symbol files_ on software accessible to the public, and keep the private symbol files for internal use. 

The **Debugging Tools for Windows** available from the [Windows 10 SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk/) make it possible to download correct symbol files on demand from a Microsoft symbol server, which stores symbol files for many different builds of a given executable file. The Debugging Tools for Windows is smart enough to download that symbol file for the image (an image is a specific version of an executable file) by refering to the timestamp & checksum in the executable's header. We'll eventually learn learn how view which Windows functions are being invoked when a process is run by referring to symbol files with the Sysinternals tools & Debugging Tools for Windows. 

Chapter 5: Process Monitor will dive into tracing call stacks. 

## Configuring Symbols
Sysinternals tools that use symbols require 2 parameters: the path of dbghelp.dll (one of Microsoft's debug engine DLLs) & the symbols path. dbghelp.dll is the file that provides the functionality for tracking call stacks, loading symbol files into a debugger, and resolving process memory addresses to names. 

Paths of dbghelp.dll: First is from Debugging Tools for Windows. There's also an x86 version. Second is shipped with Windows:  
* `C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\dbghelp.dll`  
* `C:\Windows\System32\dbghelp.dll`

When using Process Explorer to debug Windows, use the dbghelp.dll file that comes with the Debugging Tools for Windows to obtain the symbols file from Microsoft's symbols server. Using the built-in dbghelp.dll file will not allow the download.

The contents of the symbols path tells the debugging engine where to search for symbol files if they cannot be found in the two default locations:  
* The executable's directory. 
* The directory where the symbol file was originally created (if that directory is even listed in in the executable file in the first place)

The symbols path can consist of file-system directories & symbol-server directives. When executing for the first time, Process Monitor, VMMap, or Process Explorer will set its symbol path to the value of the `_NT_SYMBOL_PATH` environment variable. If undefined, the tool sets to `srv*https://msdl.microsoft.com/download/symbols` which uses the Microsoft public symbol server, but does not save to a local cache. [Configure the `_NT_SYMBOL_PATH` environment variable](https://docs.microsoft.com/en-us/visualstudio/profiling/how-to-specify-symbol-file-locations-from-the-command-line) with a single line in **cmd.exe**. 

The symbols path can contain both file system directories and symbol server directives at the same time, separated by a semicolon. The first element that contains symbol files is used. The format for a symbol-server directive element is `srv*DownstreamStore*SymbolServer` 

Right. An example of a symbols path could be:  
`srv*C:\Dev\MyApp\MSsymbols*https://msdl.microsoft.com/download/symbols`  
Process Explorer first searches through `C:\Dev\MyApp\MSsymbols` and if not present the website is queried. 

Another example is  
`C:\PrivateSymbols;srv*C:\MSsymbols*https://msdl.microsoft.com/download/symbols`  
And in this case the C:\PrivateSymbols directory is first searched. 

[Configuring the **dbghelp.dll** and **symbols** paths](https://i.imgur.com/A4zbBZw.png). 