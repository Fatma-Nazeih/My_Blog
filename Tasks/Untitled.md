### **Task 1: File Path Traversal (Lab 2)**

- **Vulnerability Type:** File Path Traversal / Information Disclosure.
- **Vulnerable Parameter:** `filename` (Query Parameter).
- **Discovery:** Upon inspecting the product details page, I used the **Network Tab (F12)** to analyze how images are retrieved. I identified a request to the `/image` endpoint:
    `GET /image?filename=42.jpg`
- **Exploitation:**
    The application fetches files directly from the server's filesystem without proper sanitization. By modifying the `filename` parameter to point to a sensitive system file (`/etc/passwd`), I attempted to bypass the intended directory.
    **Payload:** `/image?filename=/etc/passwd`
    ![[Pasted image 20260131012842.png]]
    
- **Evidence of Success:**
    Using **Burp Suite**, I captured the response which revealed the internal contents of the `/etc/passwd` file, confirming successful unauthorized access to server files.
    ![[Pasted image 20260131013501.png]]

---
### **Task 2: Blind OS Command Injection (Lab 3)**
- **Vulnerability Type:** Blind OS Command Injection.
- **Vulnerable Parameter:** `email` (POST Parameter in Feedback Form).
- **Vulnerability Context:** The application processes feedback details via a shell command. Because the output is not returned directly in the response (Blind), I leveraged **Output Redirection** to a writable directory (`/var/www/images/`) identified in the lab description.
- **Exploitation Strategy:**
    
    1. **Injection:** Intercepted the feedback submission in **Burp Suite Repeater**.
    2. **Payload Execution:** Injected a command into the `email` field to execute `whoami` an
	    save the result into a text file.
	    **Payload:** `email=x||whoami>/var/www/images/output.txt||`
    
    ![[Pasted image 20260131020753.png]]
    
- **Retrieving the Result:**
	To confirm execution, I requested the newly created file through the image loading endpoint discovered in Task 1:
    `GET /image?filename=output.txt`
- **Impact & Evidence:**
	The server responded with the username `peter-cIxEdV`. This confirms the ability to execute arbitrary system commands with the privileges of the web server user.
    ![[Pasted image 20260131022435.png]]
---
### **Summary of Improvements:**
- **Structured Format:** Divided into Discovery, Exploitation, and Evidence.
- **Terminology:** Used "Sanitization," "Unauthorized Access," and "Arbitrary System Commands."
- **Reduced Redundancy:** Links were replaced with descriptive text where possible.
- **Impact Focused:** Clearly stated what the result proves (Privileges of the web server user).