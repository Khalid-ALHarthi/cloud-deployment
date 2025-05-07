# Blog Application Deployment Guide

This repository contains the setup instructions for deploying a MERN stack blog application with AWS infrastructure.

## AWS S3 Configuration

### Frontend Bucket

1. Create an S3 bucket:
   - Name: `yourname-blogapp-frontend`
   - Region: `eu-north-1`
   - Uncheck: Block all public access
   - Enable: Static Website Hosting

2. Add Bucket Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::yourname-blogapp-frontend/*"
    }
  ]
}
```

### Media Bucket

1. Create an S3 bucket:
   - Name: `yourname-blogapp-media`
   - Uncheck: Block all public access

2. Add Bucket Policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicGetPutPost",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::yourname-blogapp-media/*"
        }
    ]
}
```
3. Add CORS Configuration:
```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": ["ETag"]
  }
]
```
4. Test Upoload
```bash
curl -v -X PUT -T image.jpg http://yourname-blogapp-media.s3.amazonaws.com/image.jpg
```

## IAM User Setup

1. Create a user:
   - Username: `blog-app-user`
   - Access type: Programmatic access

2. Create and attach the following policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::yourname-blogapp-media",
        "arn:aws:s3:::yourname-blogapp-media/*"
        "arn:aws:s3:::yourname-blogapp-frontend",
        "arn:aws:s3:::yourname-blogapp-frontend/*"
      ]
    }
  ]
}
```

3. Save the Access Key ID and Secret Access Key for later use

## EC2 Instance Setup

1. Launch an instance:
   - Name: `blog-backend`
   - Instance type: `t3.micro`
   - AMI: Ubuntu 22.04 LTS
   - Region: `eu-north-1`

2. Configure security group with the following inbound rules:
   - SSH: TCP 22, Source: My IP
   - HTTP: TCP 80, Source: 0.0.0.0/0
   - HTTPS: TCP 443, Source: 0.0.0.0/0
   - Custom TCP: 5000, Source: 0.0.0.0/0

3. User data script (to be run at instance launch):
```bash
#!/bin/bash
apt update -y
apt install -y git curl unzip tar gcc g++ make unzip
su - ubuntu << 'EOF'
export NVM_DIR="$HOME/.nvm"
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source "$NVM_DIR/nvm.sh"
nvm install --lts
nvm use --lts
npm install -g pm2
EOF
curl -L https://downloads.mongodb.com/compass/mongosh-2.1.1-linux-x64.tgz -o mongosh.tgz
tar -xvzf mongosh.tgz
mv mongosh-*/bin/mongosh /usr/local/bin/
chmod +x /usr/local/bin/mongosh
rm -rf mongosh*
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
rm -rf aws awscliv2.zip
```

## Application Deployment

### Repository Setup

Clone the repository:
```bash
git clone https://github.com/cw-barry/blog-app-MERN.git /home/ubuntu/blog-app
cd /home/ubuntu/blog-app/backend
```

### Backend Configuration

Create a `.env` file in the backend directory:
```bash
cat > .env << EOF
PORT=5000
HOST=0.0.0.0
MODE=production
MONGODB=mongodb+srv://test:qazqwe123@mongodb.txkjsso.mongodb.net/blog-app
JWT_SECRET=$(openssl rand -hex 32)
JWT_EXPIRE=30min
JWT_REFRESH=$(openssl rand -hex 32)
JWT_REFRESH_EXPIRE=3d
AWS_ACCESS_KEY_ID=<your-access-key>
AWS_SECRET_ACCESS_KEY=<your-secret-key>
AWS_REGION=eu-north-1
S3_BUCKET=yourname-blogapp-media
MEDIA_BASE_URL=https://yourname-blogapp-media.eu-north-1.amazonaws.com
DEFAULT_PAGINATION=20
EOF
```

### Frontend Configuration

Create a `.env` file in the frontend directory:
```bash
cd ../frontend
cat > .env << EOF
VITE_BASE_URL=http://<your-ec2-dns>:5000/api
VITE_MEDIA_BASE_URL=https://yourname-blogapp-media.eu-north-1.amazonaws.com
EOF
```

### Backend Deployment

Install dependencies and start the backend service:
```bash
cd /home/ubuntu/blog-app/backend
npm install
mkdir -p logs
pm2 start index.js --name "blog-backend"
pm2 startup
sudo pm2 startup systemd -u ubuntu
sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/*/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu
pm2 save
```

### Frontend Deployment

Build and deploy the frontend to S3:
```bash
cd /home/ubuntu/blog-app/frontend
npm install -g pnpm@latest-10
pnpm install
pnpm run build
aws configure  # enter access key, secret, region = eu-north-1
aws s3 sync dist/ s3://yourname-blogapp-frontend/
```

## Notes

- Replace `yourname` in bucket names with your actual username or unique identifier
- Replace `<your-access-key>` and `<your-secret-key>` with your actual AWS IAM credentials
- Replace `<your-ec2-dns>` with your actual EC2 instance's public DNS name
