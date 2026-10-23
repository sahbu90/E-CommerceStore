# E-Commerce Application Deployment Using Terraform and Docker

## 1. Project Overview

This project deploys a multi-service Node.js E-Commerce application on
AWS EC2. Terraform provisions the infrastructure as code, while Docker
runs the frontend, backend microservices, and MongoDB database.

## 2. Objectives

-   Provision AWS infrastructure using Terraform.
-   Create a VPC, public subnet, Internet Gateway, route table, and
    security group.
-   Launch an EC2 instance and configure it to run Docker containers.
-   Deploy a frontend, four backend services, and MongoDB.
-   Verify the website and backend health endpoints.

## 3. Technologies Used

  Technology              Purpose
  ----------------------- ---------------------------------------------------
  AWS EC2                 Virtual machine hosting the application
  Amazon VPC              Isolated virtual network
  Public subnet           Subnet with public IP assignment
  Internet Gateway        Internet connectivity for the VPC
  Route table             Routes internet-bound traffic through the gateway
  Security Group          Controls network access
  Terraform               Infrastructure as Code
  Docker                  Runs each application component in a container
  Node.js microservices   User, product, cart, and order services
  Nginx                   Serves frontend and proxies API paths
  MongoDB                 Application database

## 4. Deployment Architecture

All components run on one EC2 instance. The frontend is exposed on HTTP
port 80. Backend service ports and MongoDB are used for
container-to-container communication.

``` text
Browser
   |
 HTTP :80
   |
AWS Security Group
   |
EC2: 52.62.155.57
   |
Docker containers
   +-- Frontend / Nginx :80
   +-- User service     :3001
   +-- Product service  :3002
   +-- Cart service     :3003
   +-- Order service    :3004
   +-- MongoDB          :27017
```

Nginx serves the frontend and is configured to proxy `/user/`,
`/product/`, `/cart/`, and `/order/` paths to the respective backend
services.

> This is a learning/assignment deployment. It currently uses HTTP, not
> HTTPS, and all components run on a single EC2 instance. Production
> deployments should use HTTPS, appropriate secret management, database
> access controls, backups, monitoring, and a more resilient
> architecture.

## 5. AWS Infrastructure

The Terraform configuration deployed in AWS region `ap-southeast-2`
(Sydney) includes:

  -----------------------------------------------------------------------
  Resource                            Configuration / purpose
  ----------------------------------- -----------------------------------
  VPC                                 CIDR `10.20.0.0/16`, tag
                                      `ecommerce-vpc`

  Public subnet                       CIDR `10.20.1.0/24`, public IP
                                      assignment enabled

  Internet Gateway                    Connects the VPC to the internet

  Public route table                  Default route `0.0.0.0/0` through
                                      the Internet Gateway

  Route table association             Associates the subnet with the
                                      public route table

  Security Group                      HTTP port 80 and SSH port 22
                                      restricted to configured client
                                      CIDR

  EC2 instance                        Ubuntu 24.04 AMI, `t3.micro`, key
                                      pair `mykeypair`

  Ubuntu AMI data source              Finds a suitable Ubuntu 24.04 AMI
                                      from Canonical
  -----------------------------------------------------------------------

### Deployed resource outputs

-   **EC2 instance ID:** `i-0bb2bd3e9e544ca50`
-   **Public IP:** `52.62.155.57`
-   **VPC ID:** `vpc-068456b3fb56c1bde`
-   **Website URL:** `http://52.62.155.57`
-   **Region:** `ap-southeast-2`

SSH access is restricted through the `ssh_allowed_cidr` variable in
`terraform.tfvars`. Configure it with the administrator's public IP
followed by `/32`; do not expose SSH to everyone.

## 6. Docker Images and Containers

  ---------------------------------------------------------------------------------------
  Component               Image                                   Port / purpose
  ----------------------- --------------------------------------- -----------------------
  Frontend                `sahbu/ecommerce-frontend:1.0`          Nginx; host port 80
                                                                  published

  User service            `sahbu/ecommerce-user-service:1.0`      Container port 3001

  Product service         `sahbu/ecommerce-product-service:1.0`   Container port 3002

  Cart service            `sahbu/ecommerce-cart-service:1.0`      Container port 3003

  Order service           `sahbu/ecommerce-order-service:1.0`     Container port 3004

  MongoDB                 `mongo:7`                               Container port 27017
  ---------------------------------------------------------------------------------------

Only the frontend HTTP port is published on the EC2 host. Backend and
database ports are used inside the Docker network.

## 7. Deployment Procedure

### Step 1 --- Open the Terraform directory

On the Terraform control host:

``` bash
cd /home/ubuntu/terreaform-ecommerce
```

The directory contains Terraform configuration files, variables,
outputs, `terraform.tfvars`, and the provider lock file.

### Step 2 --- Configure SSH access

In `terraform.tfvars`, configure the trusted public IPv4 address:

``` hcl
ssh_allowed_cidr = "YOUR_PUBLIC_IP/32"
```

Replace the example with the administrator's current public IP.

### Step 3 --- Initialize Terraform

``` bash
terraform init
```

Downloads required provider plugins and prepares the working directory.

### Step 4 --- Validate configuration

``` bash
terraform validate
```

Checks Terraform syntax and configuration consistency.

### Step 5 --- Generate and review the plan

``` bash
terraform plan -no-color -input=false -out=tfplan
```

The reviewed plan reported **7 to add, 0 to change, 0 to destroy**. The
Ubuntu AMI data source is a lookup, not a created AWS resource.

### Step 6 --- Apply the plan

``` bash
terraform apply -no-color tfplan
```

Terraform created the AWS resources and EC2 instance. The EC2 user-data
script installs/starts Docker, starts MongoDB, and launches the
application containers.

### Step 7 --- Check Terraform outputs and state

``` bash
terraform output
terraform state list
```

These commands show the deployment outputs and resources tracked by
Terraform. Protect the state file because it contains infrastructure
details and may contain sensitive values.

### Step 8 --- Connect to the EC2 instance

From the administrator's machine, using the correct private key:

``` bash
ssh -i mykeypair.pem ubuntu@52.62.155.57
```

Keep the private key secure and do not commit it to Git.

### Step 9 --- Verify initialization and containers

Run on the application EC2 instance:

``` bash
sudo cloud-init status --long
sudo docker ps -a
```

The cloud-init command checks whether user-data initialization
completed. `docker ps -a` lists container states.

### Step 10 --- Check backend health endpoints

Run on the EC2 instance:

``` bash
sudo docker exec frontend wget -qO- http://user-service:3001/health
sudo docker exec frontend wget -qO- http://product-service:3002/health
sudo docker exec frontend wget -qO- http://cart-service:3003/health
sudo docker exec frontend wget -qO- http://order-service:3004/health
```

All four health checks returned successful JSON responses showing the
service name, `status: OK`, and port.

## 8. Verification Results

The following deployment checks were completed:

-   Terraform outputs showed the EC2 instance ID, public IP, VPC ID, and
    website URL.
-   `terraform state list` showed the VPC, subnet, Internet Gateway,
    route table, route table association, security group, and EC2
    instance.
-   The EC2 instance was running and SSH access succeeded.
-   Cloud-init completed with `status: done` and no reported errors.
-   `docker ps -a` showed all six containers running.
-   User, product, cart, and order health endpoints returned successful
    responses.
-   MongoDB logs showed application connections and activity involving
    the `ecommerce_orders.orders` collection.
-   The public homepage loaded in a browser at `http://52.62.155.57`.

## 9. Application Screenshot

The screenshot below was captured after the public homepage loaded
successfully.

![E-Commerce homepage running on AWS
EC2](docs/screenshots/ecommerce-homepage.png)

The page displays the E-Commerce navigation bar, Home and Products
links, Login and Register buttons, welcome banner, and Shop by Category
/ Featured Products sections.

## 10. Troubleshooting Commands

### Check containers

``` bash
sudo docker ps -a
```

### Read container logs

``` bash
sudo docker logs --tail 100 frontend
sudo docker logs --tail 100 user-service
sudo docker logs --tail 100 product-service
sudo docker logs --tail 100 cart-service
sudo docker logs --tail 100 order-service
sudo docker logs --tail 100 mongo
```

### Check EC2 status checks from the Terraform control host

``` bash
aws ec2 describe-instance-status \
  --region ap-southeast-2 \
  --instance-ids i-0bb2bd3e9e544ca50 \
  --include-all-instances \
  --query 'InstanceStatuses[0].[InstanceStatus.Status,SystemStatus.Status]' \
  --output table
```

### Check Terraform outputs again

``` bash
terraform output
```

If the homepage loads but an application feature fails, inspect the
browser developer console and logs for the relevant service. A working
homepage alone does not prove that registration, login, product, cart,
and order workflows have all been tested.

## 11. Security and Cost Considerations

-   Restrict SSH access to a trusted IP using `ssh_allowed_cidr`.
-   Never commit private keys, AWS credentials, tokens, or other
    secrets.
-   Do not commit Terraform state or saved plan files to a public
    repository.
-   Configure TLS/HTTPS before production use.
-   Apply suitable database access controls, backups, and monitoring for
    production.
-   Running AWS resources may incur charges. After collecting submission
    evidence, review the infrastructure and use `terraform destroy` only
    if you intend to delete the Terraform-managed resources. Review the
    destroy plan first and retain any required data.

## 12. Conclusion

The multi-service E-Commerce application was deployed on AWS using
Terraform and Docker. Terraform provisioned networking and compute
resources; Docker ran the frontend, four Node.js services, and MongoDB.
The public homepage was verified in a browser, and the backend health
endpoints returned successful responses.

The project demonstrates Infrastructure as Code, AWS networking, EC2
provisioning, containerized microservices, Nginx reverse proxying, and
deployment verification.

------------------------------------------------------------------------

**Submission checklist:** Add any additional evidence required by the
instructor, such as Terraform apply/output, AWS EC2 details,
`docker ps -a`, and health-check output. The homepage screenshot
included in this folder is the browser evidence currently available.
