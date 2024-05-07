>NOTE: when I write a question mark between parentheses (?) that means I there's something I don't understand and need to Google up later.   
# Administrative Rights
Standard users not in the Administrators group can have _effective_ administrative control over a computer if given the ability to configure or control software that runs in a more powerful security context like:  
* Privilages that are "admin-equivalent" like the Debug Programs, Take-Ownership, Restore, and Load Driver privileges. Not certain what these are at the moment (2021-01-11)
* Having control over systemwide file or registry locations used by admins or service accounts. Power Users group, for example, has control over certain admin-equivalent resources. A mistake from Vista days.
* Enabling the Windows Installer policy titled _AlwaysInstallElevated_ where any .msi file launched by any user runs in the context of **NT AUTHORITY\system**. Going to have to return to these concepts some day. 

> NOTE: The tool to analyze & manage these admin-equivalent rights is already built into the Windows OS: **secedit.exe**. Similar in complexity to **certutil.exe**, **dcdiag.exe**, **repadmin.exe**, and many other native executables. Definitely a subject I intend revisit (2023-08-03)

Today, end users should not be in the Administrators group. Even if a user is in Administrators, UAC executes programs with standard User rights, not Admin.  
Several Sysinternals utilities require administrative rights, several don't, and several run but with less features if the context is standard user.

Log into a computer running ≥ NT 6.0 as an Administrator (or a member of Backup Operators, Power Users, etc.) or as a standard User that has been provided "admin-equivalent" privileges. The Local Security Authority (LSA) creates 2 logon sessions for this user. Logon sessions all have a distinct <u>access token</u>.

1st token represents the user account's full rights, with all groups & privileges intact. Let's call this the _full_ token. Starting a process with the full token requires UAC elevation, mediated by the Application Information (Appinfo) service. The **Runas.exe** command is still present, but it does not invoke the Appinfo service. [I don't know what that means]  
Using **Runas.exe** to start a program and specifying an Admin context will run the target program using standard User version of the account. In other words, the target program will inherit the filtered token and run in that context??  
>One extremely specific exception to this ^. Under Comp Conf\Windows Settings\Security Settings\Local Policies\Security Options there's a policy titled _User Account Control: Use Admin Approval Mode for the built-in Administrator account_ that's disabled by default. "Admin Approval Mode" is a security safeguard. Requires that all admin/admin-equiv accounts hit Yes on a UAC prompt when launching a program.  
The policy _User Account Control: Run all administrators in Admin Approval Mode_ is enabled by default but does not pertain to the built-in Administrator account. When a program is launched in the context of **.\Administrator** the full token is used. No UAC prompt arises.  
This is why UAC has never stopped you when deploying a new VM running Windows Server. The built-in Administrator account was signed in.  
UAC token filtering and "Admin Approval Mode" do not apply to the built-in Administrator account like other admin/admin-equiv accounts.  
Today the built-in Administrator account on Windows 10 is disabled by default.

2nd token is a _filtered_ token that's roughly equivalent to that of a standard User. Powerful groups disabled, powerful privileges removed. The filtered token creates the user's initial processes like **Userinit.exe** and **Explorer.exe**. Child processes inherit the filtered token. [So will disabling Admin Approval Mode set the admin accounts to use the full token when launching programs?]

UAC elevation is triggered for a new process in one of 4 ways:  
* The program file contains a _manifest_ (?) indicating it requires elevation. **Disk2VHD.exe** and **RAMMAP.exe** are examples of Sysinternals tools that contain such a manifest.  
Use **SigCheck.exe** to view an image's manifest. [Ok so now what is an "image"?]  
* Hitting "Run as Administrator"  
* Windows heuristically determines that the app is a legacy installation program. Policy enables "installer detection" by default. No reason to ever disable.  
* The app is associated with a compatibility mode or _shim_ (?) that requires elevation.
"K:\FxHT\00 Inspiron660s base config.ps1
Something very powerful to realize about the above: launching a target app with Runas.exe in the context of, say, "/user:Carl Simon" does **NOT** start the target in the context of an admin! The only ways to trigger UAC elevation are the above 4.

If a process runs with an administrative token then a subsequent child process will not require UAC elevation because the parent token is inherited. Console utilities that require admin rights do not prompt for UAC so you must run in an elevated command shell. 

Once triggered UAC elevation can be accomplished either **silently**, with a **prompt for consent**, or a **prompt for credentials**.  
* Silent UAC elevation occurs without end-user interaction. Only available if the logged-in account is a member of the built-in Administrators group. By default, in ≥ NT 6.1. silent elevation is enabled for certain Windows commands. Silent elevation can be enabled for all elevation requests through the following security policy: Comp Conf\Windows Settings\Security Settings\Local Policies\Security Options -> _User Account Control: Behavior of the elevation prompt for administrators in Admin Approval Mode_. Set to **Elevate without prompting**. In the real world you should confirm that the setting is **Prompt for consent on the secure desktop**.  
* Prompt for Consent UAC elevation flashes the Yes/No dialog box. Only available if the logged-in account is in Administrators and is set as the default mode of accomplishing a UAC elevation.  
* Prompt for Credentials UAC elevation means popping in a username & password that holds admin rights. Pretty simple. You're making it more complicated than it is.

It's also possible to auto-deny UAC elevation for standard User accounts by setting the policy _User Account Control: Behavior of the elevation prompt for standard users_ to **Automatically deny elevation requests**.

What happens when User Account Control is disabled?  
  1. the Local Security Authority does not generate filtered tokens. Apps launched by accounts in Administrators always run with admin rights.  
  2. Protected Mode of Internet Explorer is disabled, meaning that IE runs with the full rights of the logged-on user.  
  3. File and registry virtualization is turned off. Many apps on XP that required administrative rights to work on standard User accounts required file and registry virtualization turned on.  
  4. The "modern" apps on NT 6.2 and beyond do not launch if UAC is disabled.   

# Windows API
The Windows API (aka _Win32 API_) is the user-mode system programming interface to the Windows OS family. The Windows API is described in the [Windows SDK documentation](https://docs.microsoft.com/en-us/windows/apps/desktop/). The Windows API originally consisted of C-style functions only but the downside was the sheer number of functions coupled with the lack of naming consistency and logical groupings. As a result some newer API's were developed using a different API mechanism: the **Component Object Model (COM)**. COM is an API 'mechanism' originally for Office apps to communicate data between Word/Excel/ppt/etc/ docs. Embed an Excel chart inside a Word doc? COM was being used. This ability is called **Object Linking and Embedding (OLE)** and was originally implemented using an old Windows messaging mechanism called **Dynamic Data Exchange (DDE)**. DDE was inherently limited so COM was developed as a replacement.

Examples of APIs accessed through COM include DirectShow, Windows Media Foundation, DirectX, DirectComposition, Windows Imaging Component (WIC), and the Background Intelligent Transfer Service (BITS).

Under COM, clients communicate with objects (_COM server objects_) through _interfaces_. An interface is a well-defined contract with a set of logically related methods grouped under the virtual table dispatch mechamism. This is a common way for C++ compilers to implement virtual functions dispatch. What results is binary compatibility and removal of compiler name mangling issues. Consequently, it is possible to call these methods from many languages (and compilers) such as C, C++, Visual Basic, .NET languages, Delphi, and others.  
Component implementation is loaded dynamically rather than being statically linked to the client.

The term **COM server** typically refers to a Dynamic Link Library (.dll) or an executable (.exe) where the COM classes are implemented. Regarding security, COM features cross-process marshalling, threading model, and more. 

## The Windows Runtime
Windows 8 introduced a new API & supporting runtime called the Windows Runtime aka _WinRT_. WinRT consists of platform services aimed at UWP app developers. From an API perspective, WinRT is built on top of COM, adding various extensions to the base COM infrastructure. For example, complete type metadata is available in WinRT stored as WINMD files and based on the .NET metadata format. This complete type metadata in WinRT extends a similar concept in COM known as type libraries. From an API design perspective, WinRT APIs are much more cohesive than classic Windows API functions, with namespace hierarchies, consistent naming, and programmatic patterns.

The relationship between various APIs and apps isn't clear cut. Desktop apps can use a subset of the WinRT APIs while UWP apps can use a subset of Win32 & COM APIs. Like usual, refer to the MSDN documentation for which APIs are available for each app platform. At the basic binary level, the WinRT API is still based on top of the legacy Windows binaries and APIs, even though the availability of certain APIs may not be documented or supported. WinRT is not a new "native" API for the system. Much like .NET, WinRT still leverages the traditional Win32 API.

Apps written in C++, C# (or other .NET languages), and JavaScript can consume WinRT APIs easily thanks to language projections developed for those platforms. C++ has the non-standard extension C++/CX. The normal COM interop layer for .NET allows any .NET language to consume WinRT APIs naturally and simply just as if it were pure .NET. JavaScript has the WinJS extension to access WinRT APIs. HTML must still be used to build the app's UI. 

## The .NET Framework
My machine has .NET 4.8 and 3.5. 2 major components to the .NET Framework:
1. **The Common Language Runtime (CLR)**. The CLR is the run-time engine for .NET. The CLR includes a Just In Time (JIT) compiler that translates Common Intermediate Language (CIL) instructions to the underlying hardware CPU machine language, a garbage collector, typer verification, code access security, and more. The CLR is implemented as a COM in-process server (aka DLL) and uses various facilities provided by the Win32 API.  
2. **The .NET Framework Class Library (FCL)**. The .NET FCL is a large collection of types that implement functionality typically needed by client and server apps like user interface services, networking, database access, and more. 

The .NET Framework also offers high-level programming languages like C#, Visual Basic, and F#. Combine that with the CLR, FCL, and supporting tools, the .NET Framework improves developer productivity and safety & reliability within the apps that target the .NET Framework.

[Relationship between the components of the .NET Framework](https://images1.programmersought.com/156/46/460c9ce6d9e3599a6b96de0d6f0de40c.png)

## Services, Functions, and Routines
In the context of programming documentation the word _service_ means either a callable routine in the OS, a device driver, or a server process. Here is some vocab that Windows Internals 7th Ed. uses:
* **Windows API functions**: documented, callable subroutines in the Windows API. Examples include CreateProcess, CreateFile, GetThreadContext, GetMessage. 
* **Native system services (aka _system calls_)**: the undocumented, underlying services that are callable from user mode. Top secret stuff. For example, NtCreateUserProcess is the internal system service the CreateProcess function calls to create a new process.   
* **Kernel support functions (aka _routines_)**: subroutines that can only be called from kernel mode. ExAllocatePoolWithTag is the routine that device drivers call to allocate memory from the Windows system heaps (called _pools_)
* **Windows services** A 'Windows service' is a process started by the Windows service control manager. The familiar print spooler is an example of a Windows service. However, remember that a "service" in the developer context is a callable routine, device driver, or a server process. 
* **Dynamic Link Libraries (DLLs)**: DLLs are callable subroutines linked together as a binary file that can be dynamically loaded by apps that use the subroutines. So an app loads a .dll file and then can execute the subroutines (DLLs) therein. Examples are the C run-time library msvcrt.dll and kernel32.dll, one of the Windows API subsystem libraries. Windows user-mode components & apps use DLLs extensively. The advantage that DLLs provide over static libraries is that apps share DLLs, and Windows ensures there's only 1 in-memory copy of a DLL's code among the apps that are referencing it. Note that library .NET assemblies are compiled as DLLs but without any unmanaged exported subroutines. Instead, the CLR parses compiled metadata to access the corresponding types and members. 