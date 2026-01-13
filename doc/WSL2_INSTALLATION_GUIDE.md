# DDA Installation Guide for Windows with WSL2

This guide provides step-by-step instructions for installing and running the Defect Detection Application (DDA) on Windows using WSL2 with Docker Desktop.

## Prerequisites

Before starting, ensure you have:

- **Windows 10 version 2004+** or **Windows 11**
- **WSL2 enabled** (see [Microsoft's WSL2 installation guide](https://docs.microsoft.com/en-us/windows/wsl/install))
- **Docker Desktop for Windows** with WSL2 backend (already configured per your setup)
- **At least 8GB RAM** and **64GB free disk space**
- **Administrator access** on Windows

## Table of Contents

1. [Verify WSL2 and Docker Setup](#step-1-verify-wsl2-and-docker-setup)
2. [Install Ubuntu in WSL2](#step-2-install-ubuntu-in-wsl2)
3. [Install System Dependencies](#step-3-install-system-dependencies)
4. [Clone DDA Repository](#step-4-clone-dda-repository)
5. [Install Python 3.9](#step-5-install-python-39)
6. [Install AWS CLI](#step-6-install-aws-cli)
7. [Configure Docker Integration](#step-7-configure-docker-integration)
8. [Setup Options](#step-8-choose-your-setup-path)
   - [Option A: Local Development/Testing (No AWS)](#option-a-local-developmenttesting-recommended-for-wsl2)
   - [Option B: Full AWS Greengrass Deployment](#option-b-full-aws-greengrass-deployment)

---

## Step 1: Verify WSL2 and Docker Setup

Open **PowerShell** as Administrator and verify your setup:

```powershell
# Check WSL version
wsl --list --verbose

# Expected output should show WSL version 2
# NAME      STATE           VERSION
# * Ubuntu  Running         2
```

Open **Docker Desktop** and verify:
- Settings → General → "Use the WSL 2 based engine" is **checked**
- Settings → Resources → WSL Integration → Your Ubuntu distribution is **enabled**

---

## Step 2: Install Ubuntu in WSL2

If you don't already have Ubuntu installed in WSL2:

### In PowerShell (Administrator):

```powershell
# Install Ubuntu 22.04 (recommended)
wsl --install -d Ubuntu-22.04

# Or Ubuntu 20.04
wsl --install -d Ubuntu-20.04

# Set as default
wsl --set-default Ubuntu-22.04
```

Launch Ubuntu from the Start Menu and complete the initial setup (create username and password).

---

## Step 3: Install System Dependencies

Open your **WSL2 Ubuntu terminal** (you can launch it from Windows Terminal or the Ubuntu app):

```bash
# Update package list
sudo apt update && sudo apt upgrade -y

# Install essential build tools
sudo apt install -y \
    git \
    curl \
    wget \
    unzip \
    zip \
    ca-certificates \
    gnupg \
    lsb-release \
    software-properties-common \
    build-essential

# Install GStreamer (required for video processing)
sudo apt install -y \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-libav \
    gstreamer1.0-tools \
    gstreamer1.0-x \
    gstreamer1.0-alsa \
    gstreamer1.0-gl \
    gstreamer1.0-gtk3 \
    gstreamer1.0-qt5 \
    gstreamer1.0-pulseaudio

# Install Java (required for AWS Greengrass if you plan to use it)
sudo apt install -y default-jdk

# Verify Java installation
java -version
```

---

## Step 4: Clone DDA Repository

```bash
# Navigate to your home directory
cd ~

# Clone the repository
git clone https://github.com/aws-samples/defect-detection-application.git

# Navigate into the repository
cd defect-detection-application

# Verify the clone
ls -la
```

---

## Step 5: Install Python 3.9

DDA requires Python 3.9. Install it using the deadsnakes PPA:

```bash
# Add Python PPA
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# Install Python 3.9 and development tools
sudo apt install -y \
    python3.9 \
    python3.9-venv \
    python3.9-dev \
    python3-pip

# Set Python 3.9 as the default python3
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.9 1

# Verify installation
python3 --version
# Should output: Python 3.9.x

# Upgrade pip
python3.9 -m pip install --upgrade pip

# Install required Python packages
python3.9 -m pip install --force-reinstall requests==2.32.3
python3.9 -m pip install protobuf
```

---

## Step 6: Install AWS CLI

If you plan to use AWS services (for full deployment):

```bash
# Download AWS CLI
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Unzip and install
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version

# Configure AWS credentials (if deploying to AWS)
aws configure
# Enter your AWS Access Key ID, Secret Access Key, region, and output format
```

**Note:** For local development/testing only, you can skip AWS CLI configuration.

---

## Step 7: Configure Docker Integration

Since you already have Docker Desktop with WSL2 backend, verify Docker works in WSL2:

```bash
# Test Docker
docker --version
docker ps

# Test Docker Compose
docker compose version

# If you get permission errors, add your user to the docker group
# (This may already be done by Docker Desktop integration)
sudo usermod -aG docker $USER

# Apply group changes (or logout/login)
newgrp docker

# Test again without sudo
docker ps
```

**Important:** In WSL2 with Docker Desktop, you don't need to install Docker separately. Docker Desktop automatically integrates with WSL2.

---

## Step 8: Choose Your Setup Path

You have two options for running DDA on WSL2:

### Option A: Local Development/Testing (Recommended for WSL2)

This option runs DDA locally using Docker Compose without AWS Greengrass. Perfect for development, testing, or learning.

#### 8A.1: Create Required Directories

```bash
# Create DDA data directories
sudo mkdir -p /aws_dda/dda_data
sudo mkdir -p /aws_dda/dda_triton/triton_model_repo
sudo mkdir -p /aws_dda/image-capture
sudo mkdir -p /aws_dda/inference-results

# Set permissions (replace 'your-username' with your WSL username)
sudo chown -R $USER:$USER /aws_dda
sudo chmod -R 755 /aws_dda
```

#### 8A.2: Configure Environment

```bash
cd ~/defect-detection-application/src

# Create environment file
cat > .env << 'EOF'
OS=ubuntu
DDA_SYSTEM_USER_ID=$(id -u)
DDA_SYSTEM_GROUP_ID=$(id -g)
DDA_ADMIN_USER_ID=$(id -u)
DDA_ADMIN_GROUP_ID=$(id -g)
JETSON_CUDA=false
JETSON_TENSORRT=false
AWS_REGION=us-east-1
EOF

# Source the environment
source .env
```

#### 8A.3: Build and Run DDA

```bash
cd ~/defect-detection-application/src

# Build the containers
docker compose --profile generic build

# Start DDA services
docker compose --profile generic up -d

# Check if containers are running
docker ps

# Expected output should show:
# - flask-app (backend)
# - react-webapp (frontend)
```

#### 8A.4: Access DDA

Once the containers are running:

1. **Open your browser** (in Windows, not WSL)
2. **Navigate to:** `http://localhost:3000`
3. **API endpoint:** `http://localhost:5000`

You should see the DDA web interface!

#### 8A.5: View Logs

```bash
# View all logs
docker compose --profile generic logs -f

# View backend logs only
docker compose logs -f backend_generic

# View frontend logs only
docker compose logs -f frontend
```

#### 8A.6: Stop DDA

```bash
cd ~/defect-detection-application/src
docker compose --profile generic down
```

---

### Option B: Full AWS Greengrass Deployment

This option sets up DDA as an AWS IoT Greengrass component for production edge deployment.

#### 8B.1: Setup IAM Roles and Policies

Follow the IAM setup instructions from the main README (Step 0).

#### 8B.2: Install Greengrass Development Kit (GDK)

```bash
# Install GDK
python3.9 -m pip install git+https://github.com/aws-greengrass/aws-greengrass-gdk-cli.git

# Add ~/.local/bin to PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify installation
gdk --version
```

#### 8B.3: Configure GDK

```bash
cd ~/defect-detection-application

# Edit gdk-config.json to set your region and S3 bucket
nano gdk-config.json

# Update the following:
# "bucket": "dda-component-YOUR-ACCOUNT-ID"
# "region": "YOUR-AWS-REGION" (e.g., "us-east-1")
```

#### 8B.4: Build Component

```bash
cd ~/defect-detection-application

# Run the build script
./gdk-component-build-and-publish.sh > logfile.log 2>&1 &

# Monitor the build
tail -f logfile.log

# This will take 20-40 minutes
```

#### 8B.5: Setup Edge Device

For testing in WSL2, you can set up Greengrass locally:

```bash
# Create required directories
sudo mkdir -p /aws_dda/greengrass/v2
sudo mkdir -p /aws_dda/image-capture
sudo mkdir -p /aws_dda/inference-results

# Download Greengrass installer
curl -s "https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-2.12.0.zip" > greengrass-2.12.0.zip
unzip greengrass-2.12.0.zip -d GreengrassInstaller

# Install Greengrass (replace placeholders with your values)
sudo -E java -Droot="/aws_dda/greengrass/v2" \
  -jar ./GreengrassInstaller/lib/Greengrass.jar \
  --aws-region YOUR-AWS-REGION \
  --thing-name YOUR-THING-NAME \
  --thing-group-name DDA_WSL2_Group \
  --component-default-user ggc_user:ggc_group \
  --provision true \
  --setup-system-service true
```

#### 8B.6: Deploy DDA Component

```bash
# Create deployment
aws greengrassv2 create-deployment \
  --target-arn "arn:aws:iot:YOUR-REGION:YOUR-ACCOUNT-ID:thing/YOUR-THING-NAME" \
  --components '{
    "aws.greengrass.Nucleus": {"componentVersion": "2.12.0"},
    "aws.edgeml.dda.LocalServer": {"componentVersion": "1.0.0"}
  }' \
  --deployment-name "DDA-WSL2-Deployment" \
  --region YOUR-AWS-REGION

# Monitor deployment logs
sudo tail -f /aws_dda/greengrass/v2/logs/greengrass.log
```

---

## Troubleshooting

### Issue: Docker commands fail with "permission denied"

**Solution:**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Issue: Cannot access http://localhost:3000

**Solutions:**

1. **Check if containers are running:**
   ```bash
   docker ps
   ```

2. **Check logs for errors:**
   ```bash
   docker compose --profile generic logs
   ```

3. **Check Windows Firewall** - ensure ports 3000 and 5000 aren't blocked

4. **Restart Docker Desktop** in Windows

### Issue: WSL2 runs out of memory

**Solution:** Create/edit `.wslconfig` in Windows:

```
# In Windows, create: C:\Users\YourUsername\.wslconfig

[wsl2]
memory=8GB
processors=4
swap=2GB
```

Then restart WSL2:
```powershell
# In PowerShell (Administrator)
wsl --shutdown
wsl
```

### Issue: "No space left on device"

**Solution:** Clean up Docker:
```bash
docker system prune -a --volumes
```

### Issue: GStreamer plugins not found

**Solution:**
```bash
# Reinstall GStreamer
sudo apt install --reinstall \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-libav
```

---

## Testing Your Installation

### Test 1: Access Web Interface

1. Open browser to `http://localhost:3000`
2. You should see the DDA dashboard

### Test 2: Check Backend API

```bash
curl http://localhost:5000/health
# Should return a health check response
```

### Test 3: Upload Test Image

1. In the DDA web interface, navigate to the Images section
2. Upload a test image
3. Verify processing completes

---

## Next Steps

After successful installation:

1. **Configure your camera sources** in the DDA web interface
2. **Train a model** using the SageMaker notebooks included in the repository
3. **Deploy your model** following the model deployment guide
4. **Set up image capture** and configure inference pipelines

---

## Additional Resources

- **Main README:** [README.md](../README.md)
- **DDA Documentation:** https://docs.aws.amazon.com/lookout-for-vision/latest/dda-user-guide/
- **WSL2 Documentation:** https://docs.microsoft.com/en-us/windows/wsl/
- **Docker Desktop WSL2:** https://docs.docker.com/desktop/wsl/

---

## Getting Help

- **GitHub Issues:** https://github.com/aws-samples/defect-detection-application/issues
- **AWS Support:** For AWS service-related questions

---

## Notes for WSL2 Users

### Performance Considerations

- Store project files in the WSL2 filesystem (`/home/...`) rather than Windows filesystem (`/mnt/c/...`) for better performance
- Docker volumes mounted from WSL2 filesystem are significantly faster

### Accessing WSL2 Files from Windows

You can access your WSL2 files in Windows Explorer:
```
\\wsl$\Ubuntu-22.04\home\your-username\defect-detection-application
```

### Running Commands

- Run DDA commands in the **WSL2 Ubuntu terminal**
- Access the web interface from **Windows browser**
- Docker Desktop manages the Docker engine for both Windows and WSL2

---

**Congratulations!** You've successfully installed DDA on Windows with WSL2.
