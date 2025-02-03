---
title: "Nullcon Goa Hack IM 2025 CTF"
date: 2025-02-02T00:00:00Z
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

# Web Craphp: CRC Collision Challenge Walkthrough

## Challenge 
![](Pasted%20image%2020250202222829.png)


## Initial Reconnaissance

The challenge presents a login form accessible through a web interface:

![](Pasted%20image%2020250202222846.png)


The source code, accessible via the `?source` parameter, reveals a PHP-based application:

```php
 <?php
ini_set("error_reporting", 0);
ini_set("display_errors",0);

if(isset($_GET['source'])) {
    highlight_file(__FILE__);
}


// https://www.php.net/manual/en/function.crc32.php#28012
function crc16($string) {
  $crc = 0xFFFF;
  for ($x = 0; $x < strlen ($string); $x++) {
    $crc = $crc ^ ord($string[$x]);
    for ($y = 0; $y < 8; $y++) {
      if (($crc & 0x0001) == 0x0001) {
        $crc = (($crc >> 1) ^ 0xA001);
      } else { $crc = $crc >> 1; }
    }
  }
  return $crc;
}


// https://stackoverflow.com/questions/507041/crc8-check-in-php/73305496#73305496
function crc8($input)
{
$crc8Table = [
    0x00, 0x07, 0x0E, 0x09, 0x1C, 0x1B, 0x12, 0x15,
    0x38, 0x3F, 0x36, 0x31, 0x24, 0x23, 0x2A, 0x2D,
    0x70, 0x77, 0x7E, 0x79, 0x6C, 0x6B, 0x62, 0x65,
    0x48, 0x4F, 0x46, 0x41, 0x54, 0x53, 0x5A, 0x5D,
    0xE0, 0xE7, 0xEE, 0xE9, 0xFC, 0xFB, 0xF2, 0xF5,
    0xD8, 0xDF, 0xD6, 0xD1, 0xC4, 0xC3, 0xCA, 0xCD,
    0x90, 0x97, 0x9E, 0x99, 0x8C, 0x8B, 0x82, 0x85,
    0xA8, 0xAF, 0xA6, 0xA1, 0xB4, 0xB3, 0xBA, 0xBD,
    0xC7, 0xC0, 0xC9, 0xCE, 0xDB, 0xDC, 0xD5, 0xD2,
    0xFF, 0xF8, 0xF1, 0xF6, 0xE3, 0xE4, 0xED, 0xEA,
    0xB7, 0xB0, 0xB9, 0xBE, 0xAB, 0xAC, 0xA5, 0xA2,
    0x8F, 0x88, 0x81, 0x86, 0x93, 0x94, 0x9D, 0x9A,
    0x27, 0x20, 0x29, 0x2E, 0x3B, 0x3C, 0x35, 0x32,
    0x1F, 0x18, 0x11, 0x16, 0x03, 0x04, 0x0D, 0x0A,
    0x57, 0x50, 0x59, 0x5E, 0x4B, 0x4C, 0x45, 0x42,
    0x6F, 0x68, 0x61, 0x66, 0x73, 0x74, 0x7D, 0x7A,
    0x89, 0x8E, 0x87, 0x80, 0x95, 0x92, 0x9B, 0x9C,
    0xB1, 0xB6, 0xBF, 0xB8, 0xAD, 0xAA, 0xA3, 0xA4,
    0xF9, 0xFE, 0xF7, 0xF0, 0xE5, 0xE2, 0xEB, 0xEC,
    0xC1, 0xC6, 0xCF, 0xC8, 0xDD, 0xDA, 0xD3, 0xD4,
    0x69, 0x6E, 0x67, 0x60, 0x75, 0x72, 0x7B, 0x7C,
    0x51, 0x56, 0x5F, 0x58, 0x4D, 0x4A, 0x43, 0x44,
    0x19, 0x1E, 0x17, 0x10, 0x05, 0x02, 0x0B, 0x0C,
    0x21, 0x26, 0x2F, 0x28, 0x3D, 0x3A, 0x33, 0x34,
    0x4E, 0x49, 0x40, 0x47, 0x52, 0x55, 0x5C, 0x5B,
    0x76, 0x71, 0x78, 0x7F, 0x6A, 0x6D, 0x64, 0x63,
    0x3E, 0x39, 0x30, 0x37, 0x22, 0x25, 0x2C, 0x2B,
    0x06, 0x01, 0x08, 0x0F, 0x1A, 0x1D, 0x14, 0x13,
    0xAE, 0xA9, 0xA0, 0xA7, 0xB2, 0xB5, 0xBC, 0xBB,
    0x96, 0x91, 0x98, 0x9F, 0x8A, 0x8D, 0x84, 0x83,
    0xDE, 0xD9, 0xD0, 0xD7, 0xC2, 0xC5, 0xCC, 0xCB,
    0xE6, 0xE1, 0xE8, 0xEF, 0xFA, 0xFD, 0xF4, 0xF3
];

    $byteArray = unpack('C*', $input);
    $len = count($byteArray);
    $crc = 0;
    for ($i = 1; $i <= $len; $i++) {
        $crc = $crc8Table[($crc ^ $byteArray[$i]) & 0xff];
    }
    return $crc & 0xff;
}

$MYPASSWORD = "AdM1nP@assW0rd!";
include "flag.php";

if(isset($_POST['password']) && strlen($MYPASSWORD) == strlen($_POST['password'])) {
    $pwhash1 = crc16($MYPASSWORD);
    $pwhash2 = crc8($MYPASSWORD);

    $password = $_POST['password'];
    $pwhash3 = crc16($password);
    $pwhash4 = crc8($password);

    if($MYPASSWORD == $password) {
        die("oops. Try harder!");
    }
    if($pwhash1 != $pwhash3) {
        die("Oops. Nope. Try harder!");
    }
    if($pwhash2 != $pwhash4) {
        die("OoOps. Not quite. Try harder!");
    }
    $access = true;
 
    if($access) {
        echo "You win a flag: $FLAG";
    } else {
        echo "Denied! :-(";
    }
} else {
    echo "Try harder!";
}
?>

<html>
    <head>
        <title>Craphp</title>
    </head>
    <body>
        <h1>Craphp</h1>
        <form action="/" method="post">
            <label for="password">Give me your password!</label><br>
            <input type="text" name="password"><br>
            <button type="submit">Submit</button>
        </form>
        <p>To view the source code, <a href="/?source">click here.</a>
    </body>
</html>
Try harder!
Craphp
Give me your password!

To view the source code, click here. 
```

## Technical Analysis

The source code analysis reveals several critical components:

1. Hardcoded password: `AdM1nP@assW0rd!`
2. Two hash functions: 
   - CRC16 (custom implementation)
   - CRC8 (with lookup table)
3. Input validation logic

### Challenge Requirements

The flag retrieval requires a password input that meets the following criteria:
1. Length must match `AdM1nP@assW0rd!` (15 characters)
2. Must NOT equal `AdM1nP@assW0rd!`
3. Must generate identical CRC16 hash
4. Must generate identical CRC8 hash

## Exploitation Strategy

The vulnerability stems from using CRC (Cyclic Redundancy Check) algorithms for password validation. These algorithms, designed for error detection rather than security, are susceptible to hash collisions.

### Solution Approach

The exploitation utilizes character permutations of the original password. This method proves effective because:
1. Maintains the required length constraint
2. Utilizes the existing character set
3. Exploits CRC's sensitivity to character ordering

### Solution Implementation

The following Go code systematically generates and tests permutations:

```go
package main

import (
	"fmt"
)

const originalPassword = "AdM1nP@assW0rd!"

// CRC8 table from the PHP code
var crc8Table = []byte{
	0x00, 0x07, 0x0E, 0x09, 0x1C, 0x1B, 0x12, 0x15,
	0x38, 0x3F, 0x36, 0x31, 0x24, 0x23, 0x2A, 0x2D,
	0x70, 0x77, 0x7E, 0x79, 0x6C, 0x6B, 0x62, 0x65,
	0x48, 0x4F, 0x46, 0x41, 0x54, 0x53, 0x5A, 0x5D,
	0xE0, 0xE7, 0xEE, 0xE9, 0xFC, 0xFB, 0xF2, 0xF5,
	0xD8, 0xDF, 0xD6, 0xD1, 0xC4, 0xC3, 0xCA, 0xCD,
	0x90, 0x97, 0x9E, 0x99, 0x8C, 0x8B, 0x82, 0x85,
	0xA8, 0xAF, 0xA6, 0xA1, 0xB4, 0xB3, 0xBA, 0xBD,
	0xC7, 0xC0, 0xC9, 0xCE, 0xDB, 0xDC, 0xD5, 0xD2,
	0xFF, 0xF8, 0xF1, 0xF6, 0xE3, 0xE4, 0xED, 0xEA,
	0xB7, 0xB0, 0xB9, 0xBE, 0xAB, 0xAC, 0xA5, 0xA2,
	0x8F, 0x88, 0x81, 0x86, 0x93, 0x94, 0x9D, 0x9A,
	0x27, 0x20, 0x29, 0x2E, 0x3B, 0x3C, 0x35, 0x32,
	0x1F, 0x18, 0x11, 0x16, 0x03, 0x04, 0x0D, 0x0A,
	0x57, 0x50, 0x59, 0x5E, 0x4B, 0x4C, 0x45, 0x42,
	0x6F, 0x68, 0x61, 0x66, 0x73, 0x74, 0x7D, 0x7A,
	0x89, 0x8E, 0x87, 0x80, 0x95, 0x92, 0x9B, 0x9C,
	0xB1, 0xB6, 0xBF, 0xB8, 0xAD, 0xAA, 0xA3, 0xA4,
	0xF9, 0xFE, 0xF7, 0xF0, 0xE5, 0xE2, 0xEB, 0xEC,
	0xC1, 0xC6, 0xCF, 0xC8, 0xDD, 0xDA, 0xD3, 0xD4,
	0x69, 0x6E, 0x67, 0x60, 0x75, 0x72, 0x7B, 0x7C,
	0x51, 0x56, 0x5F, 0x58, 0x4D, 0x4A, 0x43, 0x44,
	0x19, 0x1E, 0x17, 0x10, 0x05, 0x02, 0x0B, 0x0C,
	0x21, 0x26, 0x2F, 0x28, 0x3D, 0x3A, 0x33, 0x34,
	0x4E, 0x49, 0x40, 0x47, 0x52, 0x55, 0x5C, 0x5B,
	0x76, 0x71, 0x78, 0x7F, 0x6A, 0x6D, 0x64, 0x63,
	0x3E, 0x39, 0x30, 0x37, 0x22, 0x25, 0x2C, 0x2B,
	0x06, 0x01, 0x08, 0x0F, 0x1A, 0x1D, 0x14, 0x13,
	0xAE, 0xA9, 0xA0, 0xA7, 0xB2, 0xB5, 0xBC, 0xBB,
	0x96, 0x91, 0x98, 0x9F, 0x8A, 0x8D, 0x84, 0x83,
	0xDE, 0xD9, 0xD0, 0xD7, 0xC2, 0xC5, 0xCC, 0xCB,
	0xE6, 0xE1, 0xE8, 0xEF, 0xFA, 0xFD, 0xF4, 0xF3,
}

// CRC16 implementation from the PHP code
func crc16(s string) uint16 {
	crc := uint16(0xFFFF)
	for i := 0; i < len(s); i++ {
		crc = crc ^ uint16(s[i])
		for j := 0; j < 8; j++ {
			if (crc & 0x0001) == 0x0001 {
				crc = ((crc >> 1) ^ 0xA001)
			} else {
				crc = crc >> 1
			}
		}
	}
	return crc
}

// CRC8 implementation from the PHP code
func crc8(s string) byte {
	var crc byte
	for i := 0; i < len(s); i++ {
		crc = crc8Table[(crc^byte(s[i]))&0xff]
	}
	return crc & 0xff
}

// Check if a string is a valid collision
func isValidCollision(original, test string) bool {
	if original == test {
		return false
	}
	if crc16(original) != crc16(test) {
		return false
	}
	if crc8(original) != crc8(test) {
		return false
	}
	return true
}

// Swap characters at positions i and j in a byte slice
func swap(data []byte, i, j int) {
	data[i], data[j] = data[j], data[i]
}

// Algorithm for generating permutations
func generatePermutations(data []byte, n int, results chan<- string) {
	if n == 1 {
		test := string(data)
		if isValidCollision(originalPassword, test) {
			results <- test
			return
		}
		return
	}

	for i := 0; i < n; i++ {
		generatePermutations(data, n-1, results)
		if n%2 == 1 {
			swap(data, 0, n-1)
		} else {
			swap(data, i, n-1)
		}
	}
}

func main() {
	fmt.Printf("Original password: %s\n", originalPassword)
	fmt.Printf("Length: %d\n", len(originalPassword))
	fmt.Printf("CRC16: 0x%04x\n", crc16(originalPassword))
	fmt.Printf("CRC8: 0x%02x\n\n", crc8(originalPassword))

	// Channel for results
	results := make(chan string)

	// Start permutation generation in a goroutine
	go func() {
		data := []byte(originalPassword)
		fmt.Println("Generating permutations...")
		generatePermutations(data, len(data), results)
		results <- "" // No collision found
	}()

	// Wait for result
	if collision := <-results; collision != "" {
		fmt.Printf("\n[!] Found collision!\n")
		fmt.Printf("Original: %s\n", originalPassword)
		fmt.Printf("Collision: %s\n", collision)
		fmt.Printf("CRC16: 0x%04x\n", crc16(collision))
		fmt.Printf("CRC8: 0x%02x\n", crc8(collision))
	} else {
		fmt.Println("\n[-] No collision found")
	}
}
```

### Execution Results

Running the solution program produces a valid collision:

![](Pasted%20image%2020250202223022.png)

## Flag Retrieval

Submitting the generated collision string to the web form results in successful flag capture:

![](Pasted%20image%2020250202223103.png)

## Security Recommendations
1. Avoid non-cryptographic hash functions in security contexts
2. Implement proper cryptographic hash functions
3. Design robust authentication mechanisms


# Web ZONEy: DNS Challenge Walkthrough

## Challenge
![](Pasted%20image%2020250202224405.png)

## Initial Reconnaissance

The challenge immediately indicates DNS involvement through:
- UDP service on port 5007 (non-standard DNS port)
- Domain name reference in "zoney.eno"
- Emphasis on "ZONE" in the name

### Initial DNS Enumeration
Basic DNS enumeration on the service revealed a functioning DNS server:

```bash
dig @52.59.124.14 -p 5007 zoney.eno

# Response shows:
# - SOA record
# - NS records: ns1.zoney.eno and ns2.zoney.eno
# - Everything points to 127.0.0.1
```

## Technical Analysis

### DNS Record Discovery
MX record query revealed an additional subdomain:

```bash
dig @52.59.124.14 -p 5007 -t MX zoney.eno

# Response showed:
zoney.eno.      7200    IN      MX      10 challenge.zoney.eno.
```

### NSEC Record Investigation
NSEC record query for the discovered subdomain exposed the flag location:

```bash
dig @52.59.124.14 -p 5007 NSEC challenge.zoney.eno

# Critical response:
challenge.zoney.eno.    86400   IN      NSEC    hereisthe1337flag.zoney.eno. A RRSIG NSEC
```

## Solution Implementation

The solution path followed a logical DNS enumeration process:
1. Initial zone reconnaissance
2. Discovery of additional subdomains through MX records
3. NSEC record query revealing hidden domains
4. Flag retrieval from the exposed domain

Final flag retrieval command:
```bash
dig @52.59.124.14 -p 5007 TXT hereisthe1337flag.zoney.eno +short
```

![](Pasted%20image%2020250202224707.png)
## Reference
- [`DNS Record Types`](https://en.wikipedia.org/wiki/List_of_DNS_record_types)