# Backend Deployment Assessment

## Overview
This project provisions AWS infrastructure using Terraform and deploys a Golang backend API (MuchToDo) containerized with Docker, running against a MongoDB database.

## Prerequisites
- Terraform >= 1.0 installed
- Docker and Docker Compose installed
- AWS CLI configured with IAM credentials
- Git installed
- An AWS account with appropriate permissions
- A key pair for SSH access

## Project Structurebackend-deployment-assessment/

├── terraform/

│   ├── main.tf

│   ├── variables.tf

│   ├── outputs.tf

│   ├── terraform.tfvars.example

│   └── user_data/

│       ├── backend_setup.sh

│       └── mongodb_setup.sh

├── Dockerfile

├── docker-compose.yml

├── .dockerignore

├── evidence/

└── README.md## Phase 1: Infrastructure Provisioning (Terraform)

### Resources Created
- VPC with CIDR 10.0.0.0/16 named startuptech-vpc
- 2 Public Subnets in different availability zones
- 2 Private Subnets in different availability zones
- Internet Gateway attached to the VPC
- NAT Gateway in a public subnet
- Route tables and subnet associations
- Security Groups: ALB, Bastion, Backend, MongoDB
- EC2 Instances: Bastion (public), Backend (private), MongoDB (private)
- Application Load Balancer with target group on port 8080
- Health check configured at /health endpoint

### Steps to Provision Infrastructure

#### Step 1: Clone the repository
```bash
git clone https://github.com/Favio-1/backend-deployment-assessment.git
cd backend-deployment-assessment
```

#### Step 2: Configure AWS CLI
```bash
aws configure
```
Enter your AWS Access Key ID, Secret Access Key, region (us-east-1), and output format (json).

#### Step 3: Set up Terraform variables
```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
nano terraform.tfvars
```
Fill in your values:aws_region    = "us-east-1"

instance_type = "t3.micro"

my_ip         = "YOUR_IP_ADDRESS"

key_name      = "YOUR_KEY_PAIR_NAME"
#### Step 4: Initialize Terraform
```bash
terraform init
```

#### Step 5: Preview infrastructure changes
```bash
terraform plan
```

#### Step 6: Apply infrastructure
```bash
terraform apply
```
Type `yes` when prompted. This takes about 5 minutes.

#### Step 7: Note the outputs
After apply completes, note these values:
- alb_dns_name
- bastion_public_ip
- backend_private_ip
- mongodb_private_ip
- vpc_id

## Phase 2: Docker Setup

### Environment Variables
| Variable | Description | Default |
|----------|-------------|---------|
| PORT | Application port | 8080 |
| MONGO_URI | MongoDB connection string | mongodb://mongodb:27017 |
| DB_NAME | Database name | muchtodo |
| JWT_SECRET_KEY | JWT signing key | - |
| JWT_EXPIRATION_HOURS | JWT expiry in hours | 72 |
| ENABLE_CACHE | Enable Redis cache | false |
| LOG_LEVEL | Logging level | info |
| LOG_FORMAT | Log format | json |

### Local Development

#### Step 1: Clone the application repository
```bash
git clone -b feature/backend-only https://github.com/Innocent9712/much-to-do.git app
cd app/Server/MuchToDo
```

#### Step 2: Create environment file
```bash
cat > .env << 'EOF'
PORT=8080
MONGO_URI=mongodb://mongodb:27017
DB_NAME=muchtodo
JWT_SECRET_KEY=supersecretjwtkey123
JWT_EXPIRATION_HOURS=72
ENABLE_CACHE=false
LOG_LEVEL=info
LOG_FORMAT=json
EOF
```

#### Step 3: Build the Docker image
```bash
docker build -t muchtodo-backend .
```

#### Step 4: Start containers with Docker Compose
```bash
docker-compose up -d
```

#### Step 5: Verify the application is running
```bash
docker-compose ps
curl http://localhost:8080/health
```
Expected response: `{"cache":"disabled","database":"ok"}`

### Deployment to EC2

#### Step 1: SSH into Bastion host
```bash
ssh -i your-key.pem ec2-user@<bastion_public_ip>
```

#### Step 2: SSH into Backend server from Bastion
```bash
ssh -i ~/.ssh/your-key.pem ec2-user@<backend_private_ip>
```

#### Step 3: Clone the application repository
```bash
git clone -b feature/backend-only https://github.com/Innocent9712/much-to-do.git
cd much-to-do/Server/MuchToDo
```

#### Step 4: Create environment file
```bash
cat > .env << 'EOF'
PORT=8080
MONGO_URI=mongodb://mongodb:27017
DB_NAME=muchtodo
JWT_SECRET_KEY=supersecretjwtkey123
JWT_EXPIRATION_HOURS=72
ENABLE_CACHE=false
LOG_LEVEL=info
LOG_FORMAT=json
EOF
```

#### Step 5: Create docker-compose.yml
Use the docker-compose.yml from this repository but replace `build: .` with:
```yaml
image: favio1234/muchtodo-backend:latest
```

#### Step 6: Start the application
```bash
docker-compose up -d
```

#### Step 7: Verify via ALB DNS
```bash
curl http://<alb_dns_name>/health
```
Expected response: `{"cache":"disabled","database":"ok"}`

## Teardown / Cleanup

To destroy all AWS resources and stop billing:

```bash
cd terraform
terraform destroy
```

Type `yes` when prompted. This will delete all resources created by Terraform.

To stop local Docker containers:
```bash
docker-compose down
```

To remove Docker volumes:
```bash
docker-compose down -v
```
