# AWS EC2 Deployment Guide - Django App

This guide provides detailed, step-by-step instructions to deploy your Django application to an AWS EC2 instance using Gunicorn and Nginx.

## 1. AWS EC2 Instance Setup

### Launch Instance
1. **AWS Console**: Navigate to EC2 -> Launch Instance.
2. **Name**: `django-app-server`
3. **AMI**: Ubuntu 24.04 LTS (Free Tier eligible).
4. **Instance Type**: `t2.micro`.
5. **Key Pair**: Create or select an existing `.pem` key.
6. **Security Group**:
   - Allow SSH (Port 22) from your IP.
   - Allow HTTP (Port 80) from anywhere.
   - Allow HTTPS (Port 443) from anywhere.

### Connect to EC2
```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

## 2. Server Preparation

Run these commands to update the system and install dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv nginx git -y
```

### Clone and Setup Project
```bash
cd ~
git clone <your-repo-url> app
cd app

# Create virtual environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Environment Variables
Create a `.env` file in the project root:
```bash
nano .env
```
Add the following (replace with your values):
```env
SECRET_KEY=your-secret-key-here
DEBUG=False
ALLOWED_HOSTS=your-ec2-ip,localhost
```

### Run Migrations
```bash
python manage.py migrate
python manage.py collectstatic --noinput
```

## 3. Configure Gunicorn (Systemd)

Create a systemd socket file:
```bash
sudo nano /etc/systemd/system/gunicorn.socket
```
Paste:
```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

Create a systemd service file:
```bash
sudo nano /etc/systemd/system/gunicorn.service
```
Paste (replacing `ubuntu` and `/home/ubuntu/app` if necessary):
```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/app
ExecStart=/home/ubuntu/app/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          django_core.wsgi:application

[Install]
WantedBy=multi-user.target
```

Start Gunicorn:
```bash
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

## 4. Configure Nginx

Create an Nginx configuration file:
```bash
sudo nano /etc/nginx/sites-available/django_app
```
Paste:
```nginx
server {
    listen 80;
    server_name your-ec2-public-ip;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/ubuntu/app;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

Enable the configuration:
```bash
sudo ln -s /etc/nginx/sites-available/django_app /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
```

## 5. Testing

- **Public IP**: Visit `http://your-ec2-public-ip/api/hello/` (or your specific endpoint).
- **Checks**: 
  - `sudo journalctl -u gunicorn` for logs if it fails.
  - `sudo tail -f /var/log/nginx/error.log` for Nginx errors.

## 6. Optional: API Gateway (Wait for EC2 Test First)

1. **API Gateway** -> HTTP API -> Build.
2. **Integration**: HTTP -> `http://your-ec2-public-ip/`.
3. **Routes**: `ANY /{proxy+}` mapped to the integration.
4. **Stage**: `$default`.
