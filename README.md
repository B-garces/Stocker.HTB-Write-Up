# Stocker.HTB-Write-Up
Write up Report for the HackTheBox Machine Stocker
Stocker HTB Writeup

- Started with a Nmap scan `nmap -sV -sC 10.129.87.173`

![image](https://user-images.githubusercontent.com/61332852/226727064-2a9bfc88-936a-4366-a85d-42f4d4ad5f62.png)

- We get the basic two ports open `SSH` and `HTTP`. I first checked to see if there was any vulnerabilities with the services running first but nothing jumped out yet. So I went to enumerate the http server running. But first added the IP to my hosts file. 

![image](https://user-images.githubusercontent.com/61332852/226728036-fb25bf70-d75f-48ec-af3e-0556a0e99245.png)


- After exploring the website the only interesting thing I was able to find was the name of the head of the IT department Angoose which maybe can come in handy later on in the machine.  

![image](https://user-images.githubusercontent.com/61332852/226728119-f5be5d09-3c30-4939-a343-21ac1dcfe3c0.png)


- I then started the enumeration process for subdomains `gobuster vhost -u http://stocker.htb -w /usr/share/SecLists/Discovery/DNS/subdomains-topmillion-5000.txt`. Thus finding dev.stocker.htb 

![image](https://user-images.githubusercontent.com/61332852/226728245-29a27347-35b4-4899-bf3e-87bcbc14bc3f.png)


- Adding the new sub domain to the hosts file `sudo nano /etc/hosts`
 
![image](https://user-images.githubusercontent.com/61332852/226728689-fb45bed0-5580-4350-bc5a-6a46adc81627.png)
 

- The new sub domain leads to a login page that I began to inspect the page to see if there was anything interesting on the front end. To figure out how the login page was working I submitted some random credentials into the login page to see what the web traffic looked like going to the server.

![image](https://user-images.githubusercontent.com/61332852/226729130-9d88ef0a-466b-46d4-b632-270bd2702b36.png)


- Since I saw the response was looking for something in JS I changed the content type in the request to `application/json` and the login parameters to attempt a NoSQL injection using default credentials `{"username":{"$ne":"admin"}, "password":{"$ne":"pass"}}` to bypass the login to see if we get a different response from the server. 

![image](https://user-images.githubusercontent.com/61332852/226729345-eb55e8e9-35c3-404c-8dc8-b04489043589.png)


- We get a response that the request is trying to redirect to a new directory of the domain called `/stock`. Bringing us to a inventory purchasing page. Which allows us to add items to a cart submitting the purchase and getting a PDF returned to us documenting our purchase we had made.  

![image](https://user-images.githubusercontent.com/61332852/226729554-4b5388b2-d2aa-46c6-b30d-1fac90164ca0.png)

![image](https://user-images.githubusercontent.com/61332852/226729596-18817d36-bd9a-4617-a8d6-4203efd08846.png)


- I then checked the request/response in BurpSuite to check what was happening when clicking on submit purchase within the window.  

![image](https://user-images.githubusercontent.com/61332852/226729700-53f38a85-d23a-456f-bc97-9bbf8687d90d.png)


- After doing some research on the software that was executing the PDF file I find that it was vulnerable to XSS and allows us to exploit it by making it run a script to read a local file on the web sever running. https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting/server-side-xss-dynamic-pdf

- Reading some of the information on the hacktricks website we see that if we insert a iframe or embed tag within the title field it will print the results within the PDF after executing it. Using `<embed src=file:///etc/passwd></embed>` it prints us the results of the passwd file.

![image](https://user-images.githubusercontent.com/61332852/226730201-f663270e-9e01-4e53-8f50-9a9423edab19.png)

- I noticed it wasn’t showing all of it properly so I changed the script to change the size as well to display it properly `<embed height=1000px width=1000px src=file:///etc/passwd></embed>`

![image](https://user-images.githubusercontent.com/61332852/226730331-e42e9482-e596-4b3a-8b78-c2ed4eb2976d.png)


- After messing with the file paths I checked the local path that the webapp was being hosted on to display the main JS file `index.js` that was being presented within the directory. Using `<embed height=1000px width=1000px src=file:///var/www/dev/index.js></embed>` it shows some interesting things.  

![image](https://user-images.githubusercontent.com/61332852/226733584-d746c2bb-b3fa-41c7-b6c7-5b70041a0d72.png)


- We see that we have a password for a user on the system that we can start using to try to SSH into one of the users behind the web application. Since we saw very early on that `Angoose` was the head of the IT department and one of the users that showed after displaying the `passwd` file we can try his username first with the password we found. After exploring the users home folder we find our first flag.

![image](https://user-images.githubusercontent.com/61332852/226730781-29be338c-d6ae-4bc1-9922-7d2daf3af9cb.png)
 

- Now I continue enumerating to get the root flag of the machine with `sudo -l` 

![image](https://user-images.githubusercontent.com/61332852/226730967-cb106ab5-86bf-43f5-870c-6cb56fdc028d.png)

- We now see we can run `/usr/bin/node /usr/local/scripts/*.js` as root which we can start exploring options to exploit as well. Since we can run any Java Script file that we make within that path we can start making a JS file containing a exploit to become as the user root. I then went to https://gtfobins.github.io/ to see what we can do within the node binary to spawn as root.  

![image](https://user-images.githubusercontent.com/61332852/226731501-7ede49a4-9f86-4a5c-9d6f-6888ed4476f9.png)


- We now have the JS code to implement the exploit so lets go create the JS file to run.  

![image](https://user-images.githubusercontent.com/61332852/226731598-e14962c5-bcf7-4014-bd54-5392916c01a1.png)


- After running `sudo /usr/bin/node /usr/local/scripts/../../../home/angoose/exploit.js` we spawn as the root user. Thus giving us the root flag of the machine. 

![image](https://user-images.githubusercontent.com/61332852/226731705-39a5bf44-bd17-4b1d-9eaf-54c5672ae670.png)
 
