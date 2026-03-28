# Assignment 3 — Hybrid Auto-Scaling: Local VM to AWS Cloud Bursting

**Course:** CSL7510 — Virtualization and Cloud Computing  
**Student:** Anshul Kumar (M25AI2036)  
**Instructor:** Prof. Sumit Kalra  

## What This Does

When the local VirtualBox VM's CPU usage exceeds 75% for 30 seconds, a Python monitoring script automatically launches an AWS EC2 instance and deploys the same FastAPI application there. This demonstrates **cloud bursting** — offloading workload from local infrastructure to the cloud when local resources are exhausted.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Local Machine (Host)                   │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │          VirtualBox Ubuntu VM                      │  │
│  │                                                    │  │
│  │  ┌──────────────┐    ┌──────────────────────────┐  │  │
│  │  │  FastAPI App  │    │  monitor_and_scale.py    │  │  │
│  │  │  (port 8000)  │    │  - checks CPU every 10s  │  │  │
│  │  │              │    │  - if >75% for 30s        │  │  │
│  │  └──────────────┘    │  - launches EC2 via boto3 │  │  │
│  │                      └────────────┬─────────────┘  │  │
│  └───────────────────────────────────┼────────────────┘  │
└──────────────────────────────────────┼───────────────────┘
                                       │
                                       │ boto3 API call
                                       ▼
                    ┌──────────────────────────────────┐
                    │         AWS Cloud (us-east-1)     │
                    │                                   │
                    │  ┌─────────────────────────────┐  │
                    │  │  EC2 Instance (t3.micro)     │  │
                    │  │  Amazon Linux 2023           │  │
                    │  │  ┌───────────────────────┐   │  │
                    │  │  │  Same FastAPI App      │   │  │
                    │  │  │  (port 8000)           │   │  │
                    │  │  │  auto-installed via     │   │  │
                    │  │  │  user-data script       │   │  │
                    │  │  └───────────────────────┘   │  │
                    │  └─────────────────────────────┘  │
                    └──────────────────────────────────┘
```

## Project Structure

```
assignment3/
├── app/
│   └── app.py                  # FastAPI microservice (runs locally)
├── monitor/
│   └── monitor_and_scale.py    # CPU monitor + AWS trigger script
├── aws/
│   └── user_data.sh            # Bootstrap script for EC2 instance
├── docs/
│   └── launch_details.json     # Auto-generated on EC2 launch
├── requirements.txt
└── README.md
```

---

## Step-by-Step Execution Guide

### Prerequisites (Already Done from Assignments 1 & 2)

- VirtualBox installed with Ubuntu VM
- AWS account with EC2 access
- Key pair `my-asg-keypair` exists
- Security group `WebServer-SG` exists

### STEP 1: Prepare the Local VM

Boot your Ubuntu VM in VirtualBox. Open a terminal.

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python and pip (if not already installed from Assignment 1)
sudo apt install -y python3 python3-pip git curl stress

# Clone or copy the project files into the VM
mkdir -p ~/assignment3
cd ~/assignment3
# (copy the project files here — use shared folder, git clone, or scp)
```

### STEP 2: Install Python Dependencies

```bash
cd ~/assignment3
pip3 install -r requirements.txt
```

If pip gives an "externally managed environment" error:
```bash
pip3 install -r requirements.txt --break-system-packages
```

### STEP 3: Configure AWS Credentials

Inside the VM, configure your AWS access:

```bash
# Install AWS CLI if not present
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
# Enter your:
#   AWS Access Key ID
#   AWS Secret Access Key
#   Default region: us-east-1
#   Default output: json

# Verify it works
aws sts get-caller-identity
```

### STEP 4: Update Security Group for Port 8000

Your existing WebServer-SG only allows port 80/443. The FastAPI app runs on port 8000, so add a rule:

**In AWS Console → EC2 → Security Groups → WebServer-SG → Edit Inbound Rules:**

| Type        | Port  | Source    | Purpose                    |
|-------------|-------|-----------|----------------------------|
| Custom TCP  | 8000  | 0.0.0.0/0 | FastAPI app access         |

(Keep all existing rules from Assignment 2.)

### STEP 5: Update Configuration in monitor_and_scale.py

Open `monitor/monitor_and_scale.py` and verify these values match your setup:

```python
AWS_REGION = "us-east-1"
AMI_ID = "ami-0f5caa1cf4417e51b"           # Amazon Linux 2023
INSTANCE_TYPE = "t3.micro"
KEY_PAIR_NAME = "my-asg-keypair"            # From Assignment 2
SECURITY_GROUP_ID = "sg-0770a5b0dff337cf7"  # WebServer-SG from Assignment 2
```

### STEP 6: Start the Local FastAPI App

Terminal 1:
```bash
cd ~/assignment3/app
python3 -m uvicorn app:app --host 0.0.0.0 --port 8000
```

Verify it works:
```bash
# From another terminal or browser
curl http://localhost:8000
# Should show: {"message": "Hello from Local VirtualBox VM", ...}

curl http://localhost:8000/stats
# Shows CPU/memory stats
```

**SCREENSHOT 1:** Browser showing the local app at `http://localhost:8000`

### STEP 7: Start the Monitoring Script

Terminal 2:
```bash
cd ~/assignment3/monitor
python3 monitor_and_scale.py
```

You should see:
```
[2026-03-28 10:00:00] [INFO] Running pre-flight checks...
[2026-03-28 10:00:00] [INFO] AWS Account: 333650975919
[2026-03-28 10:00:00] [INFO] Pre-flight checks passed!
[2026-03-28 10:00:01] [INFO] Check #0001 | CPU:  3.2% | MEM: 45.1% | Status: OK | High Count: 0/3
[2026-03-28 10:00:11] [INFO] Check #0002 | CPU:  2.8% | MEM: 45.1% | Status: OK | High Count: 0/3
```

**SCREENSHOT 2:** Monitor running with normal CPU readings

### STEP 8: Stress the Local VM (Trigger Cloud Burst)

Terminal 3:
```bash
# Install stress tool if needed
sudo apt install -y stress

# Generate CPU load — this uses all available CPU cores
stress --cpu 4 --timeout 300
```

Watch Terminal 2 (monitor). You'll see:
```
[...] Check #0010 | CPU: 98.5% | MEM: 45.2% | Status: HIGH | High Count: 1/3
[...] Check #0011 | CPU: 99.1% | MEM: 45.2% | Status: HIGH | High Count: 2/3
[...] Check #0012 | CPU: 97.8% | MEM: 45.3% | Status: HIGH | High Count: 3/3
[...] CPU has been above 75% for 30 seconds!
[...] Triggering AWS EC2 launch...
[...] ============================================================
[...] THRESHOLD EXCEEDED - LAUNCHING AWS EC2 INSTANCE
[...] ============================================================
[...] EC2 Instance Launched Successfully!
[...] Instance ID: i-0abcdef1234567890
[...] Public IP: 54.xxx.xxx.xxx
[...] App URL: http://54.xxx.xxx.xxx:8000
```

**SCREENSHOT 3:** Monitor showing HIGH status and EC2 launch log

### STEP 9: Verify the Cloud Instance

Wait ~2-3 minutes for the EC2 instance to finish bootstrapping, then:

```bash
# Check if the app is running on EC2
curl http://<EC2_PUBLIC_IP>:8000
# Should show: {"message": "Hello from AWS EC2", ...}

curl http://<EC2_PUBLIC_IP>:8000/stats
# Shows EC2 resource stats
```

**SCREENSHOT 4:** Browser showing the app running on EC2 with `"environment": "AWS EC2"`  
**SCREENSHOT 5:** AWS Console showing the new EC2 instance with tag `AutoScale-CloudBurst`

### STEP 10: Clean Up

```bash
# Terminate the EC2 instance to avoid charges
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>

# Stop the stress test (Ctrl+C in Terminal 3)
# Stop the monitor (Ctrl+C in Terminal 2)
# Stop the app (Ctrl+C in Terminal 1)
```

---

## Screenshots Checklist for Report

1. Local VM running in VirtualBox
2. FastAPI app responding on local VM (`localhost:8000`)
3. Monitoring script running with normal CPU
4. `stress` command generating CPU load
5. Monitor detecting >75% and launching EC2
6. EC2 instance visible in AWS Console
7. FastAPI app responding from EC2 (`<public_ip>:8000`)
8. Side-by-side: local app says "Local VirtualBox VM" vs EC2 app says "AWS EC2"

---

## Key Course Concepts Demonstrated

| Concept | How It's Demonstrated |
|---|---|
| Cloud Bursting | Local overload → automatic cloud deployment |
| Rapid Elasticity (NIST) | EC2 instance provisioned within minutes |
| On-Demand Self-Service (NIST) | No human interaction needed for cloud launch |
| IaaS | AWS provides compute; we control OS, app, config |
| VM Monitoring | psutil tracks CPU/memory in real-time |
| Automation | boto3 programmatically provisions infrastructure |
| User Data Bootstrap | EC2 self-configures on launch |

---

## Troubleshooting

**"AWS credentials not configured"**  
→ Run `aws configure` and enter your Access Key ID, Secret Key, region (`us-east-1`)

**EC2 launches but app is unreachable on port 8000**  
→ Check Security Group: WebServer-SG must allow inbound TCP 8000 from 0.0.0.0/0

**stress command not found**  
→ Run `sudo apt install -y stress`

**pip install fails with "externally managed environment"**  
→ Add `--break-system-packages` flag

**EC2 app takes too long to start**  
→ SSH into EC2 and check: `cat /var/log/assignment3-setup.log`
