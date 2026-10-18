CloudBridge Technologies: Highly Available Website on AWS

A static company website for CloudBridge Technologies (Cloud Engineering & Consulting), deployed on a highly available, scalable and secure AWS architecture. The site runs on CentOS EC2 instances behind an Application Load Balancer, auto-scaled across two Availability Zones, and delivered globally over HTTPS through CloudFront with a custom domain.

Architecture
                        Users
                          │
                  ┌───────▼────────┐
                  │  Route 53 DNS  │   yourdomain.com (registered at Go54)
                  └───────┬────────┘
                          │  Alias record
                  ┌───────▼────────┐
                  │   CloudFront   │   HTTPS (ACM cert, us-east-1)
                  └───────┬────────┘
                          │  Origin
                  ┌───────▼────────┐
                  │      ALB       │   HTTPS/HTTP listener
                  └───────┬────────┘
                          │
                  ┌───────▼────────┐
                  │  Target Group  │   Health checks on /
                  └───┬────────┬───┘
                      │        │
   ┌──────────────────▼─┐   ┌──▼──────────────────┐
   │ AZ-1               │   │ AZ-2                │
   │ Public Subnet A    │   │ Public Subnet B     │
   │ EC2 #1 (CentOS)    │   │ EC2 #2 (CentOS)     │
   └────────────────────┘   └─────────────────────┘
        Managed by an Auto Scaling Group
        inside a custom VPC (4 subnets, 2 AZs)
AWS Services Used
Service	Purpose
VPC (custom)	Isolated network with 4 subnets across 2 Availability Zones
EC2 (CentOS)	Web servers hosting the site
Auto Scaling Group	Keeps instances running and scales with demand
Application Load Balancer	Distributes traffic across instances in both AZs
Target Group	Registers instances and runs health checks
Route 53	DNS hosting for the custom domain
ACM	Free SSL/TLS certificates
CloudFront	CDN, HTTPS termination, caching and lower latency
Go54	Domain registrar (nameservers pointed to Route 53)
Network Layout
VPC: custom CIDR (e.g. 10.0.0.0/16) with an Internet Gateway attached
Subnets: 4 total across 2 AZs
Public Subnet A (AZ-1): hosts EC2 instance 1
Public Subnet B (AZ-2): hosts EC2 instance 2
Two additional subnets (one per AZ), reserved for private tiers such as a database
Route tables: public subnets route 0.0.0.0/0 to the Internet Gateway
Security groups:
ALB SG: allow 80/443 from the internet
EC2 SG: allow 80 only from the ALB SG, SSH (22) only from your IP
Project Structure
cloudbridge/
├── index.html          # Single-page site (Home, About, Services, Technologies, Training, Portfolio, Contact)
├── style.css           # Styling (dark cloud theme, responsive, animations)
├── script.js           # Loader, mobile nav, stat counters, scroll reveal, contact form handler
├── profile.jpeg        # Founder photo
├── flyer1.jpeg         # Training flyer
├── flyer2.jpeg         # Consulting flyer
└── assets/
    └── favicon.png
Deployment Steps
1. Network
Create the VPC and attach an Internet Gateway.
Create 4 subnets across 2 AZs (2 public, 2 private).
Create a public route table (0.0.0.0/0 → IGW) and associate it with the public subnets.
2. Web server (CentOS)

Launch a CentOS instance and prepare it (Apache shown here; adjust if you use Nginx):

bash
sudo yum update -y
sudo yum install -y httpd unzip
sudo systemctl enable --now httpd

# Upload cloudbridge.zip to the server, then:
sudo unzip cloudbridge.zip -d /var/www/html/
sudo chown -R apache:apache /var/www/html
sudo systemctl restart httpd

Verify with curl http://localhost. Once it works, create an AMI (or put the above in the Launch Template user data) so every new instance comes up serving the site automatically.

3. Load balancing and auto scaling
Create a Target Group (HTTP :80, health check path /).
Create an Application Load Balancer (internet-facing) across Public Subnet A and B, with a listener forwarding to the Target Group.
Create a Launch Template from the AMI.
Create an Auto Scaling Group across both public subnets, attached to the Target Group (desired 2, min 2, max as needed, with a CPU-based scaling policy).
4. Domain, DNS and HTTPS
Create a Route 53 hosted zone for your domain.
At Go54, replace the domain's nameservers with the 4 NS records from the hosted zone.
Request a public certificate in ACM:
For CloudFront, the certificate must be in us-east-1.
Validate with DNS (the "Create records in Route 53" button).
Create a CloudFront distribution:
Origin: the ALB DNS name
Alternate domain names (CNAMEs): yourdomain.com, www.yourdomain.com
Custom SSL certificate: the ACM cert
Viewer protocol policy: Redirect HTTP to HTTPS
In Route 53, create A (Alias) records for yourdomain.com and www pointing to the CloudFront distribution.
5. Test
Open https://yourdomain.com and confirm the padlock and the site.
Stop one instance: the site should stay up while the ASG replaces it.
Confirm both targets show healthy in the Target Group.
Configuration Notes
Contact form: index.html posts to https://formspree.io/f/your-form-id. Create a free form at formspree.io and replace your-form-id with your real ID. Until then the form shows a notice and visitors can use the WhatsApp/email links.
Domain placeholders: replace yourdomain.com everywhere above with the real domain.
Cache updates: after changing site files, invalidate CloudFront (/*) so visitors see the new version.
Security Practices
EC2 only accepts web traffic from the ALB security group
HTTPS enforced at CloudFront with an ACM certificate
SSH restricted to a trusted IP
Least-privilege IAM for anything that manages the stack
Possible Improvements
Move EC2 instances into the private subnets behind a NAT Gateway
Add AWS WAF on CloudFront
Store the site in S3 and serve through CloudFront (cheaper, no servers)
Infrastructure as Code with Terraform
CI/CD with GitHub Actions
CloudWatch alarms and dashboards for ALB and ASG metrics
Credits
Website design & content: Fayomi Adeseye Abimbola, CloudBridge Technologies
AWS infrastructure & deployment: [Your name]
License

© 2026 CloudBridge Technologies. All rights reserved.
