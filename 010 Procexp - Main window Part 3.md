# Main window 
## .NET 
Performance counters for measuring processes that use the .NET Framework version 1.1 and above. It's no exaggeration that I do not know what this is. So I'm going to skip it. 

## Process I/O
The columns available from Process I/O tab contain attributes relating to file & device I/O, including file I/O through the LANMan redirector and WebDAV redirector. (?) 

>ASIDE: A network redirector is a software component installed on a client computer that is used for accessing files and other resources (e.g. printers) on a remote system. [Documentation on network redirectors](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/what-is-a-network-redirector-) 
> 
>"The network redirector sends (or redirects) requests for file operations from local client applications to a remote server where the requests are processed. The network redirector receives responses from the remote server that are then returned to the local application. The network redirector software creates the appearance on the client system that remote files and resources are the same as local files and resources and allows them to be used and manipulated in the same ways"
>
>So I take it that a network redirector is involved when opening a .docx file from a network share. 

Process Explorer measures the number of system calls for _NtReadFile_ (I/O reads), _NtWriteFile_ (I/O writes), and _NtDeviceIoControlFile_ ("other" I/O.) Process Explorer also tracks the number of bytes associated with those system calls.  
>CONFUSION: How is this different from Disk I/O?   

The book doesn't explain what "other" I/O is but from the Windows API function _NtDeviceIoControlFile_ I'm guessing it's just control or synchronization data. Possibly metadata. Like how transfering a file at 1Gbps from PC to NAS also produces a 1.1 Mbps data stream from NAS to PC which probably consists of overhead control and/or data integrity traffic. 

Each of the three types of Process I/O (read, write, other) has 4 statistics: I/O ops (total since process launch), I/O delta operations (change in value of I/O ops since last refresh), total I/O bytes since process launch, delta I/O bytes.

**I/O History** graphical representation where <span style="color:blue;font-size:1em;">**Blue**</span> line is total throughput and <span style="color:pink;font-size:1em;">**Pink**</span> line is write traffic. 

**I/O Priority** I/O prioritization allows the I/O subsystem to distinguish between foreground processes & lower background processes. 5 levels of I/O priority in Windows: Very Low, Low, Normal, High, and Critical. Pretty much all processes are Normal. Only the memory manager is assigned Critical priority for I/O operations. Modern versions of Windows do not use the High I/O priority label.

## Process Network
It's important to switch gears and understand that we're thinking about the network activity for each _process_ so the words used to describe concepts here might be the same words used to describe different but related concepts in the world of networking.  

Process Explorer reveals the number of TCP connect, send, receive, and disconnect operations; the number of bytes of those operations (per process); and the deltas at the resolution of refresh frequency. Windows does not track cumulative statistics on these ops so the information displayed is just data collected since Process Explorer started, and not at system boot.  

The first three columns are Network Receives, Network Sends, and Network Other. Without reading further, I can be certain this is **_NOT_** describing frames (Layer 2 network PDUs.) Instead, it probably just means a completed execution of a network-related Windows API function by the process. 

These figures do **_NOT_** include file I/O through the LANMan redirector but they do include file I/O through the WebDAV redirector. 

## Process Disk
I/O to local disks but unlike the attributes of Process I/O these columns include all disk I/O, including that initiated from the kernel and file system drivers. Does not include file I/O resolved by network redirectors or by in-memory caches. So it will not track the disk I/O from a RAMdisk? 

## Process GPU
By default Process Explorer makes no distinction between the onboard GPU and the dedicated GPU. With the default settings in place Process Explorer columns will just output data on GPU resources a process is consuming. The GPU tab of the System Information (CTRL+I) window allows the selection of engines.  

**GPU Usage**: Percentage of GPU time consumed by the running process since the last refresh.  
**GPU Dedicated Bytes**: Unigine Superposition brought this up to 2.5 GB on my system. The amount of GPU-dedicated memory allocated to the process across all GPUs. Dedicated memory is exclusively reserved for GPU use.  
**GPU Committed Bytes**: The total amount of video memory allocated by the process across all GPUs. Video memory includes that from dedicated graphics cards and integrated graphics systems like Iris by Intel.
**GPU System Bytes** The amount of system memory, from the CPU/GPU shared memory pool, that is currently pinned down for exclusive use by one of the GPUs.   