# AWS Deployment Guide - Django App

This guide explains how to deploy this Django application to an AWS EC2 instance and front it with an API Gateway.

## 1. AWS EC2 Setup

### Launch Instance
1. Log in to AWS Console -> EC2 -> Launch Instance.
2. **Name**: `django-app-server`
3. **AMI**: Ubuntu 22.04 LTS (Free Tier eligible).
4. **Instance Type**: t2.micro (Free Tier).
5. **Key Pair**: Create or select an existing `.pem` key.
6. **Security Group**:
   - Allow SSH (Port 22) from your IP.
   - Allow HTTP (Port 80) from anywhere.
   - Allow Custom TCP (Port 8000) from anywhere (for initial testing).

### Server Preparation
Connect to your EC2:
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```

Run these commands on the EC2:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv nginx -y

# Clone your project or transfer files
mkdir ~/app && cd ~/app
# (Transfer your files here)

python3 -m venv venv
source venv/bin/activate
pip install django djangorestframework gunicorn

# Run migrations
python manage.py migrate
```

### Run with Gunicorn
```bash
gunicorn --bind 0.0.0.0:8000 django_core.wsgi:application
```

## 2. API Gateway Configuration

1. Go to **API Gateway** -> **Create API**.
2. Select **HTTP API** -> Build.
3. **Add Integration**:
   - Select **HTTP**.
   - **Method**: ANY.
   - **URL**: `http://your-ec2-public-ip:8000/api/hello/`
4. **Configure Routes**:
   - Resource Path: `/hello`
   - Integration: The one you just created.
5. Create and Deploy.
6. You will get an "Invoke URL" like `https://xxxx.execute-api.region.amazonaws.com/hello`.

## 3. Testing
- **Direct IP**: `http://your-ec2-ip:8000/api/hello/`
- **API Gateway**: `https://your-api-gateway-url/hello`

> [!TIP]
> For production, you should use Nginx as a reverse proxy for Gunicorn and restrict EC2 security groups to only allow traffic from the API Gateway.
