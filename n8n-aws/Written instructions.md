## What’s Included in the Setup

- ✅ EC2 instance (`t2.micro`)
    
- ✅ Dockerized n8n
    
- ✅ External PostgreSQL (Supabase pooler or RDS)
    
- ✅ Nginx reverse proxy
    
- ✅ Certbot-managed HTTPS
    
- ✅ CloudFormation for one-click provisioning
    

---

## 📥 CloudFormation Template

You can get the full YAML template in this repository 

### 🔑 Parameters You’ll Be Asked For:

|   |   |
|---|---|
|Parameter|Description|
|`KeyName`|Your EC2 SSH Key Pair|
|`DomainName`|Your subdomain (e.g. `n8n.yourdomain.com`)|
|`Email`|Email for Let's Encrypt SSL|
|`DbHost`|PostgreSQL host (e.g. Supabase pooler)|
|`DbUser`|PostgreSQL username|
|`DbPassword`|PostgreSQL password (hidden)|
|`EncryptionKey`|Long random string to encrypt credentials|

Generate your encryption key with:

```bash
openssl rand -hex 32
```

---

## 🧠 After Stack Is Created

### 🔧 1. Point Your DNS to the EC2 Instance

Add an **A record** in Cloudflare or your DNS provider:

```html
Type: A
Name: n8n
Value: <Elastic IP from stack output>
Proxy: DNS Only
```

---

### 🔐 2. SSH into EC2

```bash
ssh -i your-key.pem ubuntu@<Elastic-IP>
```

---
## 🔐 SSH Access — .pem Key Permissions (Linux & Windows)

To connect to your EC2 instance securely, you'll use your `.pem` key file. But you must set correct permissions first, or SSH will refuse to use it.

---

#### 🐧 **For Linux / macOS:**

Run this in your terminal:

```
chmod 400 n8n.pem
```


#### 🪟 **For Windows (PowerShell / Git Bash):**

Check the current permissions:

```
icacls n8n.pem
```

Remove group/user access and allow only you:

```cmd
icacls n8n.pem /inheritance:r 
icacls n8n.pem /grant:r "$($env:USERNAME):(R)"
```

 If using Git Bash or WSL, you can also run:

```
chmod 400 n8n.pem
```

---

### 🔐 3. Run Certbot for SSL

```bash
sudo certbot --nginx -d sub.yourdomain.com -m youremail@expamle.com --agree-tos --non-interactive --redirect
```
Then restart Nginx
```shell
sudo systemctl restart nginx
```
This issues your HTTPS certificate via Let’s Encrypt.

---

### 🧪 4. Test n8n

Visit:

```html
https://n8n.yourdomain.com
```

You’ll see the n8n login screen.

---

## 🧹 Helpful Commands

### 🐳 Docker

```bash
docker-compose up -d
docker-compose logs n8n
```


---

## 🔗 Watch the Full Video Tutorial


