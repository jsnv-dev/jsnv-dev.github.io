---
title: BroncoCTF 2025
date: 2025-02-17
draft: false
tags:
  - ctf
  - learning
categories:
  - ctf
keywords:
  - ctf
  - learning
---
# Miku's Autograph Challenge Walkthrough

## Challenge 
![](Pasted%20image%2020250217154311.png)

## Initial Reconnaissance

### Initial Analysis
Upon accessing the website, the source code reveals several interesting components:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Hatsune Miku Fan Club</title>
    <style>
        body { 
            font-family: Arial, sans-serif; 
            text-align: center; 
            background-color: #87CEFA; 
        }
        h1 { color: #00AEEF; }
        img { width: 300px; }
        .container { 
            background: white; 
            padding: 20px; 
            border-radius: 10px; 
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1); 
            display: inline-block;
            margin-top: 20px;
        }
        button {
            margin-top: 20px;
            padding: 10px 20px;
            font-size: 16px;
            background-color: #00AEEF;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        iframe {
            margin-top: 20px;
            width: 560px;
            height: 315px;
            border: none;
        }
    </style>
    <script>
        function magicMikuLogin() {
            fetch('/get_token')
            .then(response => response.json())
            .then(data => {
                let token = data.your_token;
                fetch('/login', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: 'magic_token=' + encodeURIComponent(token)
                })
                .then(response => response.text())
                .then(result => document.body.innerHTML = result);
            });
        }
    </script>
</head>
<body>
    <h1>Welcome to Hatsune Miku's Fan Club</h1>
    
    <h3>Registered Users: miku_user & miku_admin</h3>
    <div class="container">
       <button onclick="magicMikuLogin()">✨ Magic Miku Login ✨</button>
    </div>
    <br>
    <div class="container">
    
        <iframe src="https://www.youtube.com/embed/LaEgpNBt-bQ" allowfullscreen></iframe>
    
    </div>
    <br>
    <div class="container">
        <h3>Not logged in </h3>
    </div>
</body>
</html>
```

The HTML code shows key features that warrant attention:
1. A "Magic Miku Login" button that triggers the `magicMikuLogin()` function
2. A clearly displayed list of registered users: `miku_user` and `miku_admin`
3. Client-side JavaScript that handles the authentication flow

The JavaScript code reveals a two-step authentication process:
1. Fetch a token from `/get_token`
2. Submit the token to `/login` for authentication

## Technical Analysis

### Authentication Flow Investigation
When clicking the Magic Miku Login button, the application makes a request to `/get_token`:

![](Pasted%20image%2020250217154720.png)

The response contains a JWT token. This token becomes particularly interesting as it reveals the authentication mechanism. Looking at the subsequent login request:

![](Pasted%20image%2020250217154810.png)

The login response shows a critical piece of information: access is denied because the current user is not miku_admin. This suggests that the objective is to gain administrative access.

### JWT Token Analysis
Examining the obtained token using a JWT decoder reveals its structure:

![](Pasted%20image%2020250217154845.png)

The decoded token exposes several important details:
1. The token uses the `HS256` algorithm for signing
2. The payload contains a `sub` claim set to `miku_user`

This structure suggests that user authorization is determined by the `sub` claim, making it a potential target for manipulation.

## Exploitation Strategy

### Token Manipulation Process
Using CyberChef as our token manipulation tool, the following modifications were made:

![](Pasted%20image%2020250217154947.png)

The token manipulation process involved:
1. Changing the algorithm from "HS256" to "none" to bypass signature verification
2. Modifying the "sub" claim from "miku_user" to "miku_admin"

### Successful Exploitation
Using the forged token in a login request yields success:

![](Pasted%20image%2020250217155021.png)

The response reveals the flag, confirming successful exploitation of the JWT vulnerability. This proves that the application accepts unsigned tokens and relies solely on the "sub" claim for authorization.

## Key Takeaways

### Vulnerability Analysis
This challenge highlights several critical web security concepts:
1. The dangers of trusting client-controlled JWT claims
2. The importance of proper token signature validation
3. The risks of exposing user role information in tokens

### Security Recommendations
To prevent such vulnerabilities:
1. Always validate JWT signatures using strong, unique secrets
2. Implement proper algorithm enforcement
3. Add server-side role validation beyond JWT claims

## References
1. JWT Attack Guide: https://portswigger.net/web-security/jwt
2. OWASP JWT Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html