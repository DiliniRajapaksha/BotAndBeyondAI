---
creation_date: 2025-06-17 

---
Think of CloudFormation as a tool that lets you define and set up cloud infrastructure (like servers, networks, or storage) using a text file, which AWS then automatically creates for you. 
This specific file sets up a free-tier server running **n8n** on AWS, with a static IP address and secure HTTPS access. Here’s a step-by-step explanation:

---

### **What is this file doing?**
This CloudFormation template creates an **AWS EC2 instance** (a virtual server) running **Ubuntu 22.04** with **n8n** installed. It also sets up:
- A **security group** to control network access (allowing SSH, HTTP, and HTTPS).
- An **Elastic IP** (a static public IP address) so the server’s address doesn’t change.
- **Docker** and **Docker Compose** to run n8n in a container.
- **NGINX** as a reverse proxy to handle web traffic.
- A **Let’s Encrypt SSL certificate** for secure HTTPS access.
- A connection to an external **PostgreSQL database** (e.g., Supabase or AWS RDS) for storing n8n data.

The template uses a single file (written in **YAML**) to define all these resources and their configurations.

---

### **Structure of the File**
The file is divided into key sections: **AWSTemplateFormatVersion**, **Description**, **Parameters**, **Resources**, and **Outputs**. Let’s go through each one.

#### **1. AWSTemplateFormatVersion**
```yaml
AWSTemplateFormatVersion: "2010-09-09"
```
This specifies the version of the CloudFormation template format. It’s like saying, “This file follows the rules of CloudFormation from September 2009.” It’s a standard header.

#### **2. Description**
```yaml
Description: Free-Tier n8n on EC2 (t2.micro) with HTTPS + Static IP (Elastic IP) + Dynamic AMI
```
This is a human-readable summary of what the template does. It tells you the template sets up n8n on a free-tier EC2 instance (t2.micro), with HTTPS, a static IP, and an automatically selected Ubuntu image.

#### **3. Parameters**
```yaml
Parameters:
  KeyName:
    Description: Existing EC2 KeyPair for SSH
    Type: AWS::EC2::KeyPair::KeyName
  DomainName:
    Description: Domain name pointing to this EC2 instance (e.g., n8n.example.com)
    Type: String
  Email:
    Description: Email for Let's Encrypt SSL registration
    Type: String
  UbuntuAmi:
    Description: Latest Ubuntu 22.04 LTS AMI (Auto-resolved)
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id
  EncryptionKey:
    Description: Long random string used to encrypt credentials in the DB
    Type: String
    NoEcho: true
  DbPassword:
    Description: Supabase DB password for n8n
    Type: String
    NoEcho: true
  DbHost:
    Description: Hostname of your PostgreSQL server (e.g. Supabase or RDS endpoint)
    Type: String
  DbUser:
    Description: PostgreSQL username for n8n (e.g. postgres.zsaowodxmpvnkjfblavh)
    Type: String
```
**Parameters** are like input fields. When you deploy this template in AWS, you’ll be asked to provide values for these. For example:
- **KeyName**: The name of an SSH key pair (already created in AWS) to log into the server.
- **DomainName**: A domain (e.g., n8n.example.com) you’ve set up to point to this server.
- **Email**: Used for registering a free SSL certificate with Let’s Encrypt.
- **UbuntuAmi**: The ID of the Ubuntu 22.04 image to use (automatically fetched from AWS’s parameter store, so it’s always the latest).
- **EncryptionKey**, **DbPassword**, **DbHost**, **DbUser**: Credentials and connection details for a PostgreSQL database that n8n will use to store data.

The `NoEcho: true` setting hides sensitive inputs (like passwords) in the AWS console for security.

#### **4. Resources**
This is the core of the template, defining the AWS resources to create.

##### **a. N8NSecurityGroup**
```yaml
N8NSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Allow SSH, HTTP, HTTPS
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 0.0.0.0/0
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
```
This creates a **security group**, which acts like a firewall for the EC2 instance. It allows:
- **Port 22** (SSH) for remote login.
- **Port 80** (HTTP) for web traffic (before HTTPS is set up).
- **Port 443** (HTTPS) for secure web traffic.
The `CidrIp: 0.0.0.0/0` means “allow traffic from any IP address.” This is open for convenience but could be restricted for better security.

##### **b. N8NInstance**
```yaml
N8NInstance:
  Type: AWS::EC2::Instance
  Properties:
    InstanceType: t2.micro
    KeyName: !Ref KeyName
    SecurityGroups:
      - !Ref N8NSecurityGroup
    ImageId: !Ref UbuntuAmi
    UserData:
      Fn::Base64: !Sub | ...
```
This creates the **EC2 instance** (the virtual server):
- **InstanceType: t2.micro**: A free-tier, low-power server with 1 CPU and 1 GB RAM.
- **KeyName**: Uses the SSH key you specified to allow secure login.
- **SecurityGroups**: Applies the firewall rules from `N8NSecurityGroup`.
- **ImageId**: Uses the Ubuntu 22.04 image from the `UbuntuAmi` parameter.
- **UserData**: A script that runs when the instance starts. This is where most of the setup happens.

The **UserData** script (encoded in Base64 for AWS) does the following:
1. Updates the Ubuntu system and installs **Docker**, **Docker Compose**, **NGINX**, and **Certbot** (for SSL certificates).
2. Creates a directory (`/home/ubuntu/n8n`) and a `docker-compose.yml` file to run n8n in a Docker container.
3. Configures n8n with:
   - A **PostgreSQL database** connection using the provided `DbHost`, `DbUser`, and `DbPassword`.
   - An **encryption key** for securing credentials.
   - Basic authentication (username: `admin`, password: `changeme`—you should change this!).
   - The domain name and HTTPS settings for webhooks and access.
4. Sets up **NGINX** to forward web traffic to n8n (running on port 5678 inside the container).
5. Uses **Certbot** to automatically get and install an SSL certificate for HTTPS.
6. Adds the `ubuntu` user to the Docker group for easier management.

##### **c. N8NEIP**
```yaml
N8NEIP:
  Type: AWS::EC2::EIP
  Properties:
    Domain: vpc
    InstanceId: !Ref N8NInstance
```
This creates an **Elastic IP**, a static public IP address tied to the EC2 instance. Normally, EC2 instances get a new IP every time they restart, but this ensures the IP stays the same, which is crucial for DNS and HTTPS.

#### **5. Outputs**
```yaml
Outputs:
  StaticIP:
    Description: Static Elastic IP to use in your DNS A-record
    Value: !Ref N8NEIP
  SSHCommand:
    Description: SSH access command (replace .pem with your actual key file)
    Value: !Sub "ssh -i your-key.pem ubuntu@${N8NEIP}"
  AccessURL:
    Description: Your secure n8n URL
    Value: !Sub "https://${DomainName}"
```
After deployment, CloudFormation provides these outputs:
- **StaticIP**: The Elastic IP address to point your domain’s A-record to.
- **SSHCommand**: A command to SSH into the instance (e.g., `ssh -i your-key.pem ubuntu@<IP>`).
- **AccessURL**: The HTTPS URL to access n8n (e.g., `https://n8n.example.com`).

---

### **How Does It All Work Together?**
1. You deploy this template in AWS CloudFormation, providing values like your SSH key, domain name, email, and database details.
2. AWS creates:
   - A security group allowing SSH, HTTP, and HTTPS.
   - A t2.micro EC2 instance running Ubuntu 22.04.
   - An Elastic IP for a consistent address.
3. The EC2 instance runs the UserData script, which:
   - Installs Docker, NGINX, and Certbot.
   - Sets up n8n in a Docker container, connected to your PostgreSQL database.
   - Configures NGINX to route traffic to n8n.
   - Secures the domain with a Let’s Encrypt SSL certificate.
4. Once complete, you get the IP address, SSH command, and n8n URL to access your instance.

---

### **Key Concepts for Beginners**
- **CloudFormation**: Think of it as a recipe for AWS. You write what you want (servers, IPs, etc.), and AWS makes it happen.
- **EC2 Instance**: A virtual server in the cloud, like renting a computer.
- **Security Group**: A virtual firewall controlling what traffic can reach the server.
- **Elastic IP**: A fixed IP address so your server’s address doesn’t change.
- **Docker**: Runs n8n in a lightweight, isolated environment (like a virtual app container).
- **NGINX**: A web server that forwards traffic to n8n and handles HTTPS.
- **Let’s Encrypt**: A free service for SSL certificates to make your site secure (HTTPS).

---

### **What Do You Need to Use This?**
1. An **AWS account** with a free-tier eligible account.
2. An **SSH key pair** created in AWS (for logging into the EC2 instance).
3. A **domain name** (e.g., via Route 53 or another registrar) that you can point to the Elastic IP.
4. A **PostgreSQL database** (e.g., Supabase or AWS RDS) with a hostname, username, and password.
5. An **email address** for Let’s Encrypt.

---

