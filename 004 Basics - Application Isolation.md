# Application Isolation
## Mandatory Integrity Control (MIC)
Prior to Vista, any process running as a particular user could take complete control of any other process running as that same user. So if malware.exe was running in the security context of CarlS there was almost no limit to how it could change other processes running in that same context. 

Windows Vista introduced **MIC** which is a mechanism for controlling access to securable objects like processes and files. MIC uses [1] integrity levels (IL) and [2] mandatory policy to evaluate access. Security Principals and securable objects are assigned an IL to determine their level of access or protection. This enables Windows to block a lower-integrity entity from accessing & controling a higher-integrity entity. The 4 integrity levels are **Low**, **Medium**, **High**, and **System**. Elevated apps run at the High IL, normal user apps run at the Medium IL, and low-rights apps like IE in Protected Mode run at the Low integrity level.

Mandatory Integrity Control protects elevated processes and established the foundation for 'sandboxing' that many modern applications use. 

Security principals that are standard user accounts receive Medium IL while security principals that are elevated user accounts (Power Users / local admin / etc.) are assigned the High IL. 

When a security principal attempts to launch an executable file the new process is assigned the _lower_ integrity level amongst the user & executed file. This mean that new processes will never execute with an IL higher than that of the executed file that created said process. So the file doomseeker.exe when executed created the process doomseeker.exe. The IL of the doomseeker.exe process will always be less than or equal to the IL of the doomseeker.exe file. 

Securable objects that lack a defined integrity level are treated as Medium by default. 

The IL of a security principal or securable object is stored in its _integrity label_. Integrity labels are comprised of _integrity SIDs_ and a policy. The integrity SID for a security principal is stored in its _access token_ (which may contain >1 integrity SID) while the integrity SID for a securable object is stored in its _system access control list_ (SACL). The 4 integrity SIDs are:
1. **Low** (_S-1-16-4096_)
2. **Medium** (_S-1-16-8192_)
3. **High** (_S-1-16-12288_)
4. **System** (_S-1-16-16384_)

The other component of an integrity label, the policy, defines whether to allow/deny access requests from lower-IL processes to securable objects based on the "read", "write", or "execute" nature of the lower-IL process' request. The policy that Windows assigns to all process objects blocks any and all 'read' and 'write' requests from other processes assigned a lower IL. 

Again, securable objects that lack a defined integrity level are treated as Medium by default. Windows assigns a policy that blocks 'write' requests to the securable object from processes assigned the Low IL. Note that in such a case a 'Read' request from a process of Low IL to a Medium-IL securable object is not automatically blocked! 

**Process Explorer** reveals a process' integrity level and **AccessChk** reveals the integrity labels of securable objects. 

Mandatory Integrity Control was an excellent development but it still failed to resolve a major issue: MIC does not protect two processes sharing the same IL from reading/writing each other's data.   

## AppContainers
Windows 8 saw the advent of a new application model that further protected user data & privacy, secured the corproate network, protected the integrity of system resources, and provided controlled ways of an app to obtain the necessary privileges to perform as expected. Achieving these goals meant that applications needed to be highly secured from one another. This implied that new, stronger identification of apps was necessary... and that a container mechanism that restricted an app's ability to access system resources & other apps would be required. From these ideas the _AppContainer_ was born. 

The AppContainer is an extension to the Windows security model that allows processes associated with an app to be secured as a unit. The app is strongly and uniquely identified. The app's identity is incorporated into its access tokens using a new kind of security identifier. 

When a process of an app in an AppContainer requests access to a resource, the Windows security access check applies tighter rules than it does for traditional non-AppContainer processes. Access to the resource is only granted if explicitly given. No such thing as a process in an AppContainer obtaining access to the webcam, and then a child process inheriting that permission. 

Identity verification of an app was also a concern. Someone could transfer via USB an executable titled SteamSetup.exe that was actually something else. The modern app model introduced a new packaging mechanism called _AppX_ that contains all of the app's asets in a package that is digitally signed with the publisher's cert. The app's identity is comprised of the name given to the app by its publisher, an underscore, and then a hash of the publisher's identity. For example:  
* AppleInc.iTunes_12110.26.53016.0_x64__nzyj5cx40ttqa  
* Microsoft.YourPhone_1.20112.72.0_x64__8wekyb3d8bbwe  
* MicrosoftWindows.Client.CBS_cw5n1h2txyewy  
* Amazon.com.Amazon_343d40qqvtj1t  
* MAXONComputerGmbH.Cinebench_rsne5bsk8s7tj  
* Microsoft.MinecraftUWP_1.16.20102.0_x64__8wekyb3d8bbwe  

The identity of the app's package is tied to the publisher's code signing cert. Windows uses this identity to control access to resources on the system, and the processes of this app are run in an AppContainer. An AppContainer is comprised of the following:
1. AppContainer SID: S-1-**15**-**2**-XYZ The AppContainer SID is found in the process token of that app. The value of the AppContainer SID is cryptographically derived from the app's identity. The XYZ of the SID can be quite long. Use Process Explorer to inspect the Weather app.  
2. Zero or more Capability SIDs in the form of S-1-**15**-**3**-XYZ within the process token. Again, use Process Explorer to find SIDs in the form of S-1-15-3.  
3. An AppData directory within %localappdata%\Packages in which the app is allowed to store data and subdirectories where the system can store info about the app.  
Within %localappdata%\Packages\PackageID\Settings\ there's settings.dat which is a registry hive that's loaded only when the app is running. Of course, you can manually load settings.dat in regedit.exe when the app isn't running.  
4. A separate Object Manager namespace for the app  
5. A hidden, separate installation directory for the app binaries. This directory features restrictive permissions to prevent tampering with the app. (do they mean **C:\Program Files\WindowsApps**?)

### Capabilities and Brokers
An AppContainer is a sandbox technology so apps running in an AppContainer have harshly restricted access to the system. By default it can receive input when in the foreground, display pixels on screen, and can save data to its own private data stores (see #3 above) and that's it which isn't much. Providing greater access to system resources is necessary to develop an app anyone would want to use in the first place. A stopwatch app should've have access to the webcam & microphone. But the UWP WolframAlpha app apparently needs to know my location. The modern app model uses **Capabilities** and **Brokers** to provide AppContainer apps with greater rights to resources. 

**Capabilities** in this context could be Internet Client, Location, and Webcam. Apps declare the desired capabilities in the manifest of the app's AppX package. Users can in theory view in the Microsoft Store the capabilities the app needs before downloading. After download & install users can find details under ms-settings:appsfeatures or ms-settings:privacy.  
The capabilities of AppContainer apps are represented at runtime in the process' token as Capability SIDs. An AppContainer app is unable to access the microphone unless its manifest in the AppX package contained the Capability SID for Microphone. Users might need to supply additional permission for an app to exercise a capability, as in the example of Location and the Weather app. 

> "Access your Internet connection and act as a server" and "Use data stored on an external storage device" are other capabilities.

Some well-known Capability SIDs are translatable into a human-readable format. The majority of Capability SIDs are not translated. See https://i.imgur.com/XhXWmus.png. 

**Brokers** are an alternative to declared capabilities. A _broker_ is a process that runs outside of the AppContainer, typically at Medium integrity level. An AppContainer app that needs access to a protected resource can call a WinRT API function to request access through a broker. The broker clears/denies and then performs the operation on the protected resource on behalf of the AppContainer app.  
Like Captcha, brokers commonly use an authentic user gesture for clearance / denial of an AppContainer app's request for access to protected resources. For example, I need to load vacation pictures into 123 Photo Viewer. Through a Runtime Broker, the app can call a WinRT API function to display the Select Folder dialog box. The Select Folder dialog box is _outside_ the AppContainer meaning that 123 Photo Viewer and Select Folder are unable to communicate directly. (also Select Folder us running at a higher IL?) The user selects a folder, the Runtime Broker presents the folder to the app and the images within.  
>Key point is that the app's access to the resource is not granted by changing permissions on the folder/file, which would be permanent. Instead, the system duplicates a file handle opened by the Runtime Broker and... inserts into the handle table of the AppContainer app's process? Access is permanently revoked when the app closes the object handle, and can only be re-issued via the Runtime Broker.

Uhhh so in the above example does the authentic user gesture come in the form of a user selecting the directory? No idea. 

### AppContainer resources
Specifically, that `%localappdata%\Packages\{PackageID}` directory... the AppContainer's registry hive within settings.dat that's only actually loaded into the Registry when the app is running... and the app's separate Object Manager namespace. Permissions on each of these grants access to the relevant AppContainer, and denies access to all other AppContainers. Perform the following:  
```
accesschk.exe -ldq %localappdata%\Packages\Microsoft.MinecraftUWP_8wekyb3d8bbwe\AC
```
The AppContainer SID has `FILE_ALL_ACCESS` rights to some of those directories. 

The settings.dat file within the Settings directory holds the AppContainer app's private registry hive. Yes, there's a private hive for each of these apps in the form of `\REGISTRY\A\{randomGUID}`. 

AppContainer apps have their own dedicated Named Objects container in the Object Manager. The container is created when the app starts and is deleted once the app exits. Permissions on the container allow only the relevant AppContainer app to read. Noticing a pattern? When an app creates an object like a Mutant, that object instance is brought from that app's own private Named Objects container. This prevents against attacks where Process A creates objects with names that typically belong to Process B with the intent of impersonating the other process.  
Use the WinObj tool to view the Named Objects container for a running AppContainer app. Within `\Sessions\1\AppContainerNamedObjects\AppContainerSID\` the object type ALPC Port is the Named Objects container for Minecraft for Windows 10: https://i.imgur.com/uVRqSbc.png

### AppContainer access check
When a process in an AppContainer app requests access to an object, the **Windows Security Reference Monitor** (details in Windows Internals, Ch 7) performs an additional set of checks compared to non-AppContainer apps. Ordinarily, the calling process must pass the mandatory integrity check and the discretionary access check. With AppContainer apps the resource must also grant explicit access to either (1) the AppContainer SID in the calling process' token (2) one or more Capability SIDs in the caller's token, or (3) to all AppContainers.  
But even if the resource grants access to `Everyone` the access request is denied if the object's DACL does not also explicitly grant access to at least one of the above three.  
Each AppContainer process, like any process, needs access to system-wide resources. Every process needs to load `C:\Windows\System32\Ntdll.dll`, for example. Windows defines the new well-known SID **S-1-15-2-1** which stands for `APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES`. Running the following displays which security principals have Read access (and therefore can load) to the Ntdll.dll file: 
```
accesschk.exe -q C:\Windows\System32\ntdll.dll
```
## Protected Processes
MIC (Mandatory Integrity Control) is primarily to protect apps & user data from less-trustworthy apps. AppContainers are designed to protect sandbox apps from each other. However, neither MIC or AppContainers protect user processes and data from the interactive user's desktop processes that typically run at Medium IL. The solution was **protected processes** which create a barrier to protect processes not only from user processes but even from administrators. They were originally created to protect against piracy of audio/visual media by restricting the operations that could be performed on such processes (like Audiodg.exe) that handled the media. Today, protected processes primarily defend critial system processes that a. protect the system (like anti-malware processes) or b. that manage sensitive information like user credentials. 

Normally, any process possessing the **Debug Programs** privilege can obtain any access to any other process, even if the target process' security descriptor does not grant access. Examples of what a caller process with **Debug Programs** could do to another process include: read or modify the target's memory, inject code, suspend & resume threads, and terminate the process. This means that a standard user account with the **Debug Programs** privilege, and without membership in Administrators, can run programs that bypass or disable anti-malware systems and run pass-the-hash attacks by stealing credentials from **lsass.exe**. 

Protected processes change these access rules. A process holding **Debug Programs** is unable to modify a protected process. Administrators and even the System account is blocked from almost any ability to read or perform changes on a protected process. 

Windows designates a process as a protected process based on a digital signature in the process' image file and if a config setting has established that the process is to be protected. Examples of processes that Windows always considers protected without exception are the System process **ntoskrnl.exe**, Session Manager subsystem **smss.exe**, the Windows Start-Up Application **wininit.exe**, and the Service Control Manager **services.exe** (aka the Services and Controller app.) 

Processes that include that digital signature in their image but are optionally protected are **lsass.exe** and selected services such as anti-malware processes.  
>The ancestor processes of protected process are also themselves protected processes so a chain of trust is established. 

Say a caller attempts to access a target protected process. The Windows kernel grants the caller very restricted rights that do not include the ability to read/write memory or inject code. The exception is if the caller is a protected process with a higher _precedence protection_. For a further layer of security, the target protected process loads only specially-signed DLLs so that untrusted code cannot execute within the process. These restrictions also apply to the threads of a protected process. 

Protected processes are resistant to the application compatibility shim engine loading shim DLLs into the target process. 

Windows defines several types of protected processes:  
* **PsProtectedSignerAuthenticode** () 
* **PsProtectedSignerCodeGen**  
* **PsProtectedSignerAntimalware**: Microsoft Defender Antivirus Network Inspection service (**NisSrv.exe** WdNisSvc), Microsoft Defender Antivirus Service (**MsMpEng.exe** WinDefend)  
* **PsProtectedSignerLsa**  
* **PsProtectedSignerWindows**: the Delivery Optimization service (DoSvc), Appx Deployment Service (AppXSVC), Security Center (wscsvc)  
* **PsProtectedSignerWinTcb**: the System Guard Runtime Monitor Broker service **SgrmBroker.exe**, wininit.exe, smss.exe, services.exe, and both instances of csrss.exe.  

Each of these types of protected processes apply different code-signing restrictions (remember there's a digital signature in the process' image file) like which signers are authorized to sign the process' image file, which signers are authorized to sign DLLs, and the required hash algorithm. 

Each protected process type also enforce slightly different restrictions on the access rights provided to callers. A protected process of type **PsProtectedSignerAuthenticode** can grant a caller the `PROCESS_TERMINATE` right, but a protected process of type **PsProtectedSignerAntimalware** will not grant that right. 

For the purposes of caller protected processes attempting to gain access to a target protected process, Microsoft established the concept of marking the caller as either a regular protected process or a 'protected process-light.' The latter holding lower precedence... because they're light.  
>CONFUSTION: What does _precedence_ mean here? Is there any such thing like "sub-integrity levels" within the System IL? Probably not. I don't think they mean 'Priority.' Unclear. (?)   

>NOTE: When I opened the Protection column of Process Explorer it specified that the following processes were 'light': 
>> **PsProtectedSignerWinTcb-Light**: wininit.exe, smss.exe, services.exe, and both instances of csrss.exe.  
>> **PsProtectedSignerWindows-Light**: the Delivery Optimization service (DoSvc), Appx Deployment Service (AppXSVC), Security Center (wscsvc)  
>> **PsProtectedSignerAntimalware-Light** Microsoft Defender Antivirus Network Inspection service (**NisSrv.exe** WdNisSvc), Microsoft Defender Antivirus Service (**MsMpEng.exe** WinDefend)  


