# Step-by-Step ClickOps AWS - WordPress

A step-by-step guide detailing how I deployed WordPress on AWS using the AWS Console, helping me gain a deeper understanding of AWS configurations.
---

## Part 1: VPC & Networking Steps

### Step 1: Create VPC

1. Go to **VPC** → **VPCs**
2. Click **Create VPC**
   - Name: `wordpress-vpc`
   - IPv4 CIDR: `10.0.0.0/16`
3. Click **Create VPC**

![VPC Step](./screenshots/vpc-step.png)


### Step 2: Create Two Public Subnets

1. Go to **VPC** → **Subnets**
2. Click **Create subnet**
   - VPC: `wordpress-vpc`
   - Name: `public-subnet-a`
   - AZ: `us-east-2a`
   - IPv4 CIDR: `10.0.0.0/24`
3. Click **Add new subnet**
   - VPC: `wordpress-vpc`
   - Name: `public-subnet-b`
   - AZ: `us-east-2b`
   - IPv4 CIDR: `10.0.1.0/24`

### Step 3: Create Two Private Subnets

1. Click **Add new subnet**
   - VPC: `wordpress-vpc`
   - Name: `private-subnet-a`
   - AZ: `us-east-2a`
   - IPv4 CIDR: `10.0.16.0/20`
2. Click **Add new subnet**
   - VPC: `wordpress-vpc`
   - Name: `private-subnet-b`
   - AZ: `us-east-2b`
   - IPv4 CIDR: `10.0.32.0/20`

### Step 4: Enable Auto-assign Public IP

1. Select `public-subnet-a`
2. **Actions** → **Edit subnet settings**
3. Check **Enable auto-assign public IPv4 address**
4. Click **Save**
5. Do the same for `public-subnet-b`

### Step 5: Create Internet Gateway

1. Go to **VPC** → **Internet Gateways**
2. Click **Create internet gateway**
   - Name: `wordpress-igw`
3. Click **Create**
4. Select IGW → **Attach to VPC** → Select `wordpress-vpc`

### Step 6: Create NAT Gateway

1. Go to **VPC** → **NAT Gateways**
2. Click **Create NAT Gateway**
   - Subnet: `public-subnet-a`
   - Elastic IP: Click **Allocate Elastic IP**
3. Click **Create NAT Gateway**

### Step 7: Create Route Tables

**Public Route Table:**
1. Go to **VPC** → **Route Tables**
2. Click **Create route table**
   - Name: `public-rtb`
   - VPC: `wordpress-vpc`
3. Click **Create**
4. Select it → **Routes** → **Edit routes**
5. **Add route**: Destination `0.0.0.0/0`, Target: IGW
6. Go to **Subnet associations** → Associate `public-subnet-a` and `public-subnet-b`

**Private Route Table:**
1. Click **Create route table**
   - Name: `private-rtb`
   - VPC: `wordpress-vpc`
2. Click **Create**
3. Select it → **Routes** → **Edit routes**
4. **Add route**: Destination `0.0.0.0/0`, Target: NAT Gateway
5. Go to **Subnet associations** → Associate `private-subnet-a` and `private-subnet-b`

---

## Part 2: Security Groups and IAM Role

### Step 8: Create An ALB Security Group

1. Go to **EC2** → **Security Groups**
2. Click **Create security group**
   - Name: `alb-sg`
   - VPC: `wordpress-vpc`
3. **Inbound rules**:
   - HTTP (80) from `0.0.0.0/0`
   - HTTPS (443) from `0.0.0.0/0`
4. Click **Create security group**

![Security Groups](./screenshots/aws-sg.png)

### Step 9: Create EC2 Security Group

1. Click **Create security group**
   - Name: `ec2-sg`
   - VPC: `wordpress-vpc`
2. **Inbound rules**:
   - HTTP (80) from `alb-sg`
   - HTTPS (443) from `alb-sg`
   - SSH (22) from your IP (optional)
3. Click **Create security group**

### Step 10: Create RDS Security Group

1. Click **Create security group**
   - Name: `rds-sg`
   - VPC: `wordpress-vpc`
2. **Inbound rules**:
   - MySQL/Aurora (3306) from `ec2-sg`
3. Click **Create security group**

### Step 11: Create IAM Role

1. Go to **IAM** → **Roles**
2. Click **Create role**
   - Service: **EC2**
3. Add policies:
   - `AmazonS3FullAccess`
   - `AmazonSSMManagedInstanceCore`
4. Role name: `wordpress-ec2-role`
5. Click **Create role**

---

## Part 3: RDS Database

### Step 12: Create RDS Subnet Group

1. Go to **Aurora and RDS** → **Subnet groups**
2. Click **Create DB subnet group**
   - Name: `wordpress-subnet-group`
   - VPC: `wordpress-vpc`
3. Add both availability zones and both subnets: `public-subneta` and `private-subneta` with `us-east-2a` and `public-subnetb` and `private-subnetb` with `us-east-2b`
4. Click **Create**

### Step 13: Create RDS Database

1. Go to **Aurora and RDS** → **Databases**
2. Click **Create database** → **Full configuration**
   - Engine version: **MySQL 8.4.7**
   - Template: **Free tier**
   - DB identifier: `wordpress-mysql-db`
   - Master username: `admin`
   - Master password: Create strong password
   - Instance class: `db.t3.micro`
   - Storage: `20 GB`
   - VPC: `wordpress-vpc`
   - DB subnet group: `wordpress-subnet-group`
   - Publicly accessible: **No**
   - VPC security group: `rds-sg`
   - Initial database name: Leave blank
3. Click **Create database**

![RDS Step](./screenshots/RDS.png)

---

## Part 4: Target Group and Application Load Balancer

### Step 14: Create Target Group

1. Go to **EC2** → **Target Groups**
2. Click **Create target group**:
   - Name: `wordpress-tg`
   - Protocol: **HTTP**
   - Port: **80**
   - VPC: `wordpress-vpc`
   - Success codes: `200,302`
3. Click **Next** → **Create target group**

### Step 15: Create ALB

1. Go to **EC2** → **Load Balancers**
2. Click **Create load balancer** → **Application Load Balancer**
   - Name: `wordpress-alb`
   - VPC: `wordpress-vpc`
   - AZ: `us-east-2a` and `us-east-2b`
   - Subnets: `public-subnet-a` and `public-subnet-b`
   - Security group: `alb-sg`
3. Select Target group `wordpress-tg`
4. Click **create load balancer**

![Target Group](./screenshots/aws-tg.png)

![ALB Step](./screenshots/alb.png)

---

## Part 5: EC2 Instance


### Step 16: Launch EC2 Instance

1. Go to **EC2** → **Instances**
2. Click **Launch instances**
   - Name: `wordpress-server`
   - AMI: **Amazon Linux**
   - Instance type: **t3.micro**
   - VPC: `wordpress-vpc`
   - Subnet: `private-subnet-a`
   - Security group: `ec2-sg`
   - Storage: `30 GB` (gp3)
   - Advanced details → IAM instance profile: `wordpress-ec2-role`

3. Click **Launch instances**

### Step 17: Register with Target Group

1. Go to **EC2** → **Target Groups**
2. Select `wordpress-tg`
3. Go to **Targets** tab
4. Click **Register targets**
5. Select your EC2 instance
6. Click "Include as pending below" 
7. Click **Register pending targets**

![EC2 Step](./screenshots/EC2.png)

---

## Part 6: Install WordPress

### Step 18: Connect via SSM Session Manager

1. Go to **EC2** → **Instances**
2. Select your instance
3. Click **Connect** → **SSM Session Manager** → **Connect**

### Step 19: Install Apache & PHP

```bash
sudo dnf update -y
sudo dnf install -y httpd php php-mysqlnd php-gd php-mbstring php-xml php-fpm
sudo systemctl enable httpd php-fpm
sudo systemctl start httpd php-fpm

sudo nano /etc/httpd/conf.d/php-fpm.conf

<FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost"
</FilesMatch>

sudo systemctl restart httpd php-fpm
```

### Step 20: Install WordPress

```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo mv wordpress/* .
sudo rmdir wordpress
sudo rm latest.tar.gz
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
```

### Step 21: Configure WordPress

```bash
sudo cp wp-config-sample.php wp-config.php
sudo vim wp-config.php
```

Update these lines with your RDS endpoint and credentials:
```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'admin');
define('DB_PASSWORD', 'YOUR_PASSWORD');
define('DB_HOST', 'YOUR_RDS_ENDPOINT:3306');
```

### Step 22: Create WordPress Database

```bash
sudo dnf install -y php php-mysqli
sudo systemctl restart httpd

php -r "
\$mysqli = new mysqli('YOUR_RDS_ENDPOINT', 'admin', 'YOUR_PASSWORD');
if (\$mysqli->connect_error) {
    die('Connection failed: ' . \$mysqli->connect_error);
}
\$mysqli->query('CREATE DATABASE IF NOT EXISTS wordpress');
echo 'Database created successfully';
\$mysqli->close();
"
sudo systemctl restart httpd
```

---

## Part 7: S3 Bucket

### Step 23: Create S3 Bucket

1. Go to **S3** → **Buckets**
2. Click **Create bucket**
   - Name: `wordpress-media-YOUR-ID` (must be unique)
   - Region: `us-east-2`
3. Click **Create bucket**

![S3 Bucket](./screenshots/s3.png)

---

## Part 8: VPC Endpoints

### Step 24: Enable DNS Support

1. Go to **VPC** → **Your VPCs**
2. Select `wordpress-vpc`
3. **Actions** → **Edit VPC settings**
4. Check:
   - Enable DNS hostnames
   - Enable DNS resolution
5. Click **Save**

### Step 25: Create VPC Endpoints

**SSM Endpoint:**
1. Go to **VPC** → **Endpoints**
2. Click **Create endpoint**
   - Service name: `com.amazonaws.us-east-2.ssm`
   - VPC: `wordpress-vpc`
   - Subnets: `private-subnet-a`
   - Security group: `ec2-sg`
3. Click **Create endpoint**

**SSM Messages Endpoint:**
1. Repeat with service name: `com.amazonaws.us-east-2.ssmmessages`

**EC2 Messages Endpoint:**
1. Repeat with service name: `com.amazonaws.us-east-2.ec2messages`

![VPC Endpoints](./screenshots/vpc-endpoint.png)

---

## Part 9: CloudFront Distribution

### Step 26: Create CloudFront Distribution

1. Go to **CloudFront** → **Distributions**
2. Click **Create distribution**
3. **Origins**:
   - Type: ALB
   - Origin 1: ALB domain (`wordpress-alb-xxx.us-east-2.elb.amazonaws.com`)
   - Origin 2: S3 bucket domain
4. **Behaviors**:
   - Default (*) → ALB origin
   - Path pattern `/wp-content/uploads/*` → S3 origin
5. **Viewer protocol policy**: HTTP and HTTPS
6. **Allowed HTTP methods**: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
7. Click **Create distribution**

Wait for deployment (5-10 minutes).

![cloudFront](./screenshots/cloudfront.png)

![origins](./screenshots/origins.png)

![behaviors](./screenshots/behaviors.png)

---

## Part 10: Route 53 or Cloudflare DNS

### Step 27: Connect Your Domain via Route 53 or Cloudflare

1. Go to **Route 53** → **Hosted zones**
2. Select your domain
3. Click **Create record**
   - Type: **A**
   - Alias: **Yes**
   - Alias target: Select CloudFront distribution
4. Click **Create records**

1. Go to **Cloudflare** → **DNS**
2. Select your domain
3. Add a **CNAME record**
   - Name: **www** (or **@** for root domain)
   - Target: Your **CloudFront distribution** domain (e.g., d3lv1eaqawn0q8.cloudfront.net)
   - Proxy status: **Enabled (orange cloud)** to use Cloudflare caching and HTTPS
4. Ensure **SSL/TLS mode** is Full (strict) for proper end-to-end HTTPS

![Cloudflare DNS](./screenshots/cloudflare.png)

---

## Testing

1. **CloudFront domain works**: `d3lv1eaqawn0q8.cloudfront.net`
2. **Domain works**: `http://www.abdullahabdi.com` (after DNS propagates 24-48 hours)
3. **WordPress Setup**: Complete the WordPress installation wizard
4. **Admin Dashboard**: `https://www.abdullahabdi.com/wp-admin`


---

![WordPress App](./screenshots/wordpress-app.png)

WordPress application is deployed on AWS with proper networking, database, storage, and content delivery via CloudFront.