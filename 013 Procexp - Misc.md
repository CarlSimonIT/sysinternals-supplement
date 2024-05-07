# Verify Image Signatures
The version resource of an image file (or DLL file) can include Company Name, Description, Copyright, and other fields for publisher info. Of course, anyone can write "Nvidia" in the company field so there needs to be another mechanism for identity verification. From this we can make use of digital signatures and the Verify button in the Image tab. In addition to identity verification, a digital signature also confirms whether the image / DLL file has been modified. 

The Verified Signer column could read (Verified), (No signature was present in the subject)... so not the image or DLL file hasn't been signed... and (An error occurred while reading or writing to a file) which can mean multiple things like...   
1. The file has been modified since its signing
2. The signing cert has been revoked
3. The signing cert has expired, and the signature was not countersigned by a trusted timestamp server (?)
4. The signing cert does not trace its authority back to a publicly trusted root CA that's trusted on the computer. This can happen if **_Automatic Root Certificates Update_** is disabled through Group Policy.     

The name on the signing certificate and the Company Name value of the process instance are not necessarily identical. For DLL files in the Windows OS the Verified Signer is Microsoft Windows while the Company Name is Microsoft Corporation. So just keep that in mind. 

# VirusTotal analysis
[VirusTotal.com](https://www.virustotal.com) is a free web service that lets users upload files and file hashes to be analyzed by over 50 AV engines. Uploading an entire suspicious file to VirusTotal shouldn't be done lightly. A sophisticated attacker will regularly check VirusTotal.com for their malware's file hash. If you upload the suspicious .exe for analysis and obtain results the attacker will also be aware that this was done. From there, it's just a simple matter of adding some junk code to make the .exe invisible again to VirusTotal.com. When dealing with such a determined human adversary you need to proceed carefully and not clue the attacker know you're on to them. 

# System Information
Hit **CTRL + I** and there are Summary, CPU, Memory, I/O and GPU tabs. The GPU tab includes the interface that configures which engines Process Explorer includes in all its GPU performance calculations. The Summary tab displays graphs representing systemwide metrics. Pretty simple.  

### CPU tab
Can toggle on/off "Show one graph per CPU" Red means time spent executing code in kernel mode. Green does NOT NOT NOT mean time spent executing code in user mode. It means total CPU utilization as a percentage of max CPU utilization. Place the graph in "Show one graph per CPU" mode and hover the mouse over [one of the graphs](https://i.imgur.com/FN8fwO6.png). Process Explorer demarcates between CPU cores and CPU threads! The process with the highest _systemwide_ CPU utilization at that moment is reflected. In this graph I used Processor Affinity to set Zandronum to only use CPU 23. However CPU 21 was reading zandronum.exe. So Process Explorer does not track which process consumed how much processor time on a per-CPU thread basis.  

For the per-CPU graphs Process Explorer uses the less accurate time-based tracking for CPU utilization. Cycle-based data for each CPU thread is not tracked by Windows. 

### Memory tab
We see the **System Commit** and the **Physical Memory** graphs. System Commit is total amount of private bytes committed across all processes, including the paged pool. The System Commit graph is scaled against the "commit limit" which is the maximum amount of private bytes that can be committed without increasing pagefile size. The Physical Memory graph shows the amount of physical RAM in use by the system and scaled to the amount that's available to Windows. [Memory tab](https://i.imgur.com/WV0Jy7E.png).   
The lower region of the Memory tab contains memory-related metrics:

* **Commit Charge (K)** The current commit charge is the limit of private bytes at which point no more can be allocated without mapping virtual memory to the pagefile. The peak commit charge is the max commit charge since boot.
* **Physical Memory (K)** Total physical memory available to Windows (shouldn't change), available RAM that's not in use, and the sizes of the Cache, Kernel, and Driver Working Sets. 
* **Kernel Memory (K)** Paged Working Set is the amount of paged pool (in KB) that is present in RAM. Paged Virtual is the total amount of allocated paged pool, including bytes that have been swapped out to the pagefile. Paged Limit is the maximum amount of paged pool that the system will allow to be allocated. Nonpaged is the amount of allocated nonpaged pool, in KB. Nonpaged Limit is the maximum amount of nonpaged pool that can be allocated. 
* **Paging** Going down the line... the number of page faults since the previous data refresh, the number of paging I/O reads to a mapped file or the paging file, the number of writes to the paging file, and the number of writes to mapped files. 
* **Paging Lists (K)** At this point we're dipping a toe into _Windows Internals_ material. This group lists the amount of memory in the various page lists maintained by the memory manager. 

### I/O tab
Provides I/O bytes (file & device I/O throughput,) network bytes, and disk bytes. [Example of I/O tab](https://i.imgur.com/mZTbkHj.png). Self explanatory.  
