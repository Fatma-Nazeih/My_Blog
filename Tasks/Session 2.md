### Task 1: Lab 2 in Path Traversal
First you visit the shop page and click on view details button of any product, which will display all the information about that product including the photo of that product, if we clicked on F12 button to open Dev Tools in Network Tab we can see all requests happened to det all the information in that page including a request of the photo from the server which will lead us to its directory in the server and then we can bypass it with a way or another.

![[Pasted image 20260131012842.png]]

That is the normal photo path https://0a17003b030c2abd81a899fe0080006a.web-security-academy.net/image?filename=42.jpg
if we changed the parameter to /etc/passwd https://0a17003b030c2abd81a899fe0080006a.web-security-academy.net/image?filename=/etc/passwd it will lead us directly to this screen: 
![[Pasted image 20260131013055.png]]
 To know the content of that file we will use Burp Suite to see the content of the response  ![[Pasted image 20260131013501.png]]
And Done.

---
### Task 2: Lab 3 OS Command Injection 
First you visit the shop page and click on submit feedback button which will let us post a certain comment and save it in the server at this directory `/var/www/images/` so we will send a feedback and open this request on Burp Suite and send to repeater so we can edit it:
![[Pasted image 20260131020710.png]]
That is the normal flow ![[Pasted image 20260131020753.png]]
**The Exploitation Strategy:** Instead of providing a standard email, we will inject an OS command into the `email` parameter. Since this is a **blind** vulnerability (the output isn't shown in the HTTP response), we will use **output redirection** (`>`) to force the server to write the result of our command into a text file within the web-accessible directory (`/var/www/images/`).

**The Payload:** I will modify the `email` parameter to: `email=x||whoami>/var/www/images/output.txt||`
- `||`: This acts as a command separator, allowing our injected command to execute regardless of the primary command's success.
- `whoami`: The command we want to execute.
- `>/var/www/images/output.txt`: This redirects the command output into a new file named `output.txt` in the images folder.

**Retrieving the Output:** Once the request is sent, I will access the following URL to read the contents of the file: `https://[LAB-ID].web-security-academy.net/image?filename=output.txt`
![[Pasted image 20260131022011.png]]
Again access the shop page and click on F12 to open any photo path and change the parameter to output.txt 
replace this https://0aa9001f0416bb74840eaf6e008500e7.web-security-academy.net/image?filename=56.jpg
with this https://0aa9001f0416bb74840eaf6e008500e7.web-security-academy.net/image?filename=output.txt
which will display the user name `peter-cIxEdV`
![[Pasted image 20260131022435.png]]

![[Pasted image 20260131022449.png]]
