# Sessions, window stations, desktops, and window messages
In the world of Windows we refer to things like _sessions_, _session IDs_, the "console session", _session 0_, _interactive window stations_ and _noninteractive window stations_. We speak of programs running "on the same desktop." Let's learn just what the hell these all really mean.  

A **_session_** is the sum of all processes and system objects that comprise a single user's logon experience. These system objects include all windows, desktops, and windows stations. In our context a **_desktop_** is a paged pool area specific to the session. This session-specific paged pool area loads in the kernel's virtual memory space. Session-private GUI objects (e.g. start menu, task bar) are allocated from this paged pool area. 

A **_window station_** is basically a security boundary to contain desktops and processes. In the world of Remote Desktop Services (RDS) a session can encompass several window stations, and a window station can encompass several desktops.  
![](https://i.imgur.com/K3KLmBI.png)

Only the window station **Winsta0** is permitted to interact with the user at the console. Three desktops are loaded in Winsta0: the logon screen **Winlogon**, the user desktop **Default**, and **Screen-saver** which [this reference](https://techcommunity.microsoft.com/t5/ask-the-performance-team/sessions-desktops-and-windows-stations/ba-p/372473) calls Disconnect. Each of these desktops have a separate logical display. This is why the screen changes when locking a workstation. The window station is switching from the Default desktop to the Winlogon desktop, and there is no way for a user's input on the Winlogon desktop to interact with the Default desktop. 

>User Account Control is actually a switch from the Default desktop to the Winlogon desktop.  The foreground is the UAC window and the background is actually a dimmed screenshot of the Default desktop. This is informally known as the **_secure desktop_** but it's still a Winlogon desktop. The secure desktop will only restore interaction with the Default desktop once the user supplies input. 

The Winsta0 window station houses desktops that interact with the user but that's not always the case. For example, Windows services load under the Default desktop of the **Service-0x03e7$** non-interactive window station.  
>Exceptions are services that need user input. Those load in the Winsta0 window station.  

An important point is that an RDS session is not in and of itself a logon session created by the Local Security Authority. LSA logon sessions and RDS sessions are entirely different concepts so it's critical to always keep this distinction in mind. 

## Remote Desktop Services (RDS) sessions
Let's start this section off with some vocabulary that would've been really helpful if the book defined:
>**Interactive Session**: Back-and-forth dialog between user and computer. Interactive sessions in PowerShell start with Enter-PSSession.  
>**Batch Session**: Transmitting or updating an entire file. Implies a non-interactive or non-interruptible operation from beginning to end. Batch sessions in PowerShell start with Invoke-Command.  
>**Console Session** refers to the RDS session associated with the locally attached keyboard, video and mouse. A user that logs onto a gaming PC locally and interacts with the Windows 10 GUI can describe their session as the _console session_. At the same time, an RDS server with multiple users logged on via thin clients has several console sessions. The term "console session" is relative to the identity of the user. 

Remote Desktop Services supports multiple logged on interactive user sessions on a single computer. The term _interactive user session_ includes not just the everyday UI but also console sessions like WinRM/SSH/Telnet. Lose the idea that an "interactive session" implies a GUI. 

Features of Remote Desktop Services include Fast User Switching, Remote Assistance, Remote Applications Integrated Locally (RAIL, aka RemoteApps) and VM integration features. 

An important limitation of the Windows operating system, from XP to 10, is that only 1 logged on interactive session may be **connected** at once. Processes in disconnected but logged on interactive user sessions can still continue to run. However, 1 and only 1 of those logged on interactive user sessions may update a display device or receive input from the keyboard & mouse: the session that is connected. 

>Do not use the word "active" to describe a connected session because it's ambiguous. 

When the bank employee types their un/pw into Win 10 host A, and then uses SSH/Telnet/WinRM to log into Win 10 host A from somewhere else, that logon attempt will warn that the person sitting at the computer will be kicked off. Doesn't matter whether the bank employee is using a different un/pw. 

>>>I know that on this Windows 10 Pro machine I'm unable to use another computer to create an RDS session with Remote Desktop while typing this sentence...  <span style="background-color:Black;"> but does that also mean I can't SSH/Telnet/WinRM into this Windows 10 machine?! I think it does but haven't confirmed. </span>

RDS sessions are identified with the format _session N_ where N is an integer. For session 0 the Object Manager defines a global namespace. Object Manager then allocates separate local namespaces for session 1 and onward to provide isolation between the sessions. Process X in session 1 is isolated from Process Y in session 2 because they were assigned to different local namespaces. 

Session 0 uses the global namespace as its local namespace for processes in session 0. System processes and Windows services always run in RDS session 0. Keep in mind that, Windows 10 technically runs an RDS session when you're sitting in front of the machine. 

Some people may mistake the term _console session_ as a synonym of _session 0_ but nothing could be farther from the truth. All interactive user sessions... whether connected / disconnected... whether local GUI / RDS GUI / remote CLI... are assigned RDS session 1 (or more) and a dedicated local namespace from the Object Manager. Microsoft's term for the enhanced separation between system processes and user processes is _**session 0 isolation**_.

A synonym for _session 0_ is the global terminal-services session. 

>HISTORY: In Server 2003 & non-domain XP the first interactive user to log on used session 0. Consequently, the services of that user (including custom services from 3rd party apps) shared the same session namespace as System processes and Windows services. Server 2003 and non-domain XP machines only created Session 1 when another interactive user session connected.  
> Domain-joined Windows XP hosts never created session 1 because there was no such thing as multiple logged on interactive user sessions, regardless of whether the current interactive user session was disconnected but remained logged on. UserA couldn't lock their workstation and then have UserB log on. That wasn't a thing on domain-joined Windows XP machines. 

## Window Stations
Each session contains >=1 _window stations_. A window station is a securable object that contains a clipboard, an [atom table](https://docs.microsoft.com/en-us/windows/win32/dataxchg/about-atom-table), and one or more desktops. In Windows, each process is associated with 1 window station. Within a session, only the **Winsta0** window station can display a UI or accept input.  
<span style="font-family:Consolas;font-size:1.5em;">Session 0</span> contains several window stations, including Winsta0:  
![](https://i.imgur.com/K3KLmBI.png)  
For session 0, Windows generates a new, separate window station each _LSA logon session_ associated with a service. For example, service processes that run as System run in the _Service-0x0-3e7$_ window station. Service processes that run as Network Service run in the _Service-0x0-3e4$_ window station. And for some damn reason **WinObj** files session 0 window stations under `\Windows\WindowStations\` instead of `\Sessions\0\Windows\WindowStations\`:  
![](https://i.imgur.com/mpu1SIE.png)

>DETOUR: A partial definition of PsExec syntax is <span style="background-color:Black;font-family:Consolas;font-size:1em;">PsExec.exe -s -i [session#] [TargetExe][Arguments]</span>. Leaving out the **-i** switch means to run TargetExe in session 0 and therefore the _Service-0x0-3e7$_ window station. The **-s** switch sets TargetExe to run as System. 
> So the following command will run PowerShell 7 as System in session 0, and therefore in the window station labeled _Service-0x0-3e7$_.  
>><span style="background-color:Black;font-family:Consolas;font-size:1em;">PsExec.exe -s pwsh.exe</span>  
> Chapter 7 provides a complete treatment of the PsTools suite, of which PsExec is a component.

A service configured to run as System can also be configured to **Allow Service To Interact With Desktop**. If set, the service will run in the _Winsta0_ window station of session 0 instead of the _Service-0x0-3e7$_ window station. Therefore, a user logged into a Windows XP machine could then interact with the service through the display (?) and manipulate with keyboard & mouse. _Session 0 isolation_ has remediated this exploit on modern systems. 

>>>REVIEW: Remember service accounts? When a service account "logs on" part of what's happening is that an _LSA logon session_ is being created. A service account with an LSA logon session does **not** have its own RDS session like an interactive user session would. Instead the LSA logon session of the SA shares _session 0_ & has its own window station. A locally unique identifier (LUID) for each service account with an LSA logon session is incorporated into the name of that window station. 

<span style="font-family:Consolas;font-size:1.5em;">Sessions 1</span> and onward contain only the Winsta0 window station:  
![](https://i.imgur.com/xHoRbjB.png)  
![](https://i.imgur.com/LXNv6NJ.png)  

# Desktops
Each window station contains 1 or more **desktops**. A desktop in this context is a securable object with a logical display surface on which applications can render UI in the form of windows.  
>We are _not_ talking about the Desktop abstraction at the top of the Windows Explorer shell namespace. The two concepts are unrelated.  
Multiple desktops in a window station may contain UI but only 1 may display on screen. The interactive window station (Winsta0) typically contains 3 desktops: **Default**, **Screen-saver**, and **Winlogon**. 

The Default desktop is where user apps run by default. Kind of. A process in Windows is associated with a window station and each of its threads is associated with a desktop inside that window station. Theoretically, it is possible for threads in one process to be associated with multiple desktops but it's more common for all threads to run under a single desktop. Jumping ahead: the **Desktops** utility of Sysinternals allows for the creation of up to 3 more desktops in a window station. One may run apps in these desktops. Does that also apply to desktops of type Winlogon and Disconnect? (?) 

The Screen-saver desktop is where Windows runs a screen saver, if the account is secured with a password.

The Winlogon desktop, also known as the **_secure desktop_** is what displays when you hit CTRL+ALT+DEL. This could be the username & password screen leading to a connected interactive user session. Or it could be that harsh blue screen. As stated above, a UAC prompt is actually the Winlogon desktop with a screenshot of the Default desktop in the background. Windows establishes permissions on the Winlogon desktop to only allow processes running as System. This protects secure operations involving password entry. Ever notice how CTRL+V doesn't work on the password screen of a locked console session local to the user? 

Process Monitor and Process Explorer reveal which RDS session ID a process is running in. However, none of the Sysinternals utilities directly indicate that process' window station or desktop. Clues must be gathered from the output of ProcMon and ProcExp.

>I'm wondering if what I consider to be a "local" logon session is actually nothing more than a console session where console output is represented as windows (with window messages) and user input from the keyboard & mouse is translated into console input.  

## Window messages
Unlike console apps, Windows-based applications are event driven. Each thread that creates an instance of a user object of type window (review [here](https://docs.microsoft.com/en-us/windows/win32/sysinfo/object-categories)) has a queue to which messages are sent. These messages are _window messages_. These threads (aka _GUI threads_) wait for and then process window messages as they arrive. Window messages tell a window in the GUI what to do or what occurred.

Examples of what a window message tells a GUI window include "redraw yourself", "move to screen coordinates (x,y)", "close yourself", "the Enter key was pressed", "Mouse 2 was clicked when the cursor was on coordinates (x,y)", and "the user is logging off."

Window messaging is mediated by the _window manager_. Window messages originate from a source process, are sent to the window manager, and delivered to the target process. We can consider the window manager to be analogous to a router. Window messages can be sent to any GUI window from any thread running on the same desktop. Threads running on the Winlogon desktop are unable to send window messages to a GUI window that itself is running on a thread in the Default desktop. The window manager prohibits this. 

>>>ASIDE: I guess it's common to have >1 instance of ProcMon.exe running?! Say we have Instance 1 (source) and Instance 2 (target) of ProcMon.exe running. The <span style="background-color:Black;font-family:Consolas;font-size:1em;">/Terminate</span> and <span style="background-color:Black;font-family:Consolas;font-size:1em;">/WaitForIdle</span> commands of Process Monitor must be invoked from the same desktop as the target instance of ProcMon.exe. Those commands use window messaging to tell the Instance 2 of ProcMon.exe to shut itself down, and to determine that the target instance of ProcMon.exe is ready to process commands in the form of window messages. (?)

Keyboard and mouse input can be emulated with window messages! The **RegJump** Sysinternals utility and the _Jump To_ feature in Process Monitor & Autoruns do precisely this to navigate to a key in regedit.exe. Because of the levels of abstraction between a physical keypress/mouseclick and the resulting window messages received by the thread of a GUI window, it is effectively impossible for a target GUI thread to know with absolute certainty if an arriving window message resulted from a keystroke on a keyboard or if another program simply sent those window messages.

This window messaging architecture dates back to Windows 1.0, the one update being the introduction of multithreading support on 32-bit versions of Windows. It's a legacy system that will be with us forever. One negative aspect is that window objects do not have security descriptors or access control lists. This has privilege escalation written all over it. A non-admin user's program could send specially crafted window messages to the process (GUI thread) of a window object. If that process was running as System (like an antivirus program would) then it was simple to control that process. The attacker could inject code that places the non-admin user in the local Administrators group. After rebooting the computer the processes associated with the antivirus load from disk into RAM. The injected code runs and the user logs on with local admin! This major privilege escalation exploit found in August 2002 is called a _shatter attack_, and is the main reason interactive user sessions are no longer in session 0. 

User Interface Privilege Isolation (UIPI) is another layer of protection against shatter attacks on windows owned by processes holding elevated rights. Remember that the _window manager_ is like a router for window messages between threads. User Interface Privilege Isolation updated the window manager to also detect window messages that can change the target window's state. Such a window message might be "Mouse 1 was clicked on coordinates (x,y)." With UIPI, the window manager now compares the IL (integrity level) of the source and target processes when it receives a state-changing window message. UIPI blocks the message if the sending process' IL is lower than the IL of the target process. 

UIPI is the reason the processes under RegJump and other Jump To features must execute at an IL at least as high as the regedit.exe process.

Finally, if the process sending a windowing message is part of an AppContainer app then UIPI's window manager will only clear those messages if the target process that owns the window occupies the same AppContainer as the source. [Technical documentation of MIC and UIPI](https://docs.microsoft.com/en-us/previous-versions/dotnet/articles/bb625964(v=msdn.10)).