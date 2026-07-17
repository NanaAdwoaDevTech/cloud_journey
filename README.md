# Cloud Journey
My notes on learning Cloud Computing

A good reminder: https://youtu.be/U1-g2qlnTO8
This one too: https://youtu.be/vWDhQNXM4K8
Okay, got it!: https://youtu.be/gaCY4QxfSzA
Always to remember: https://youtu.be/Rqinrxtt1fg


Learning IaC from Terraform, I now understand what they mean by Declarative.
With code, it's imperative, meanining we are instructing it to get a result. Here, in declarative, you just give it the result you want. i.e. instead of wanting 5 EC2 servers from already having 2 deployed by saying 'Add 3 more EC2 servers', we just say, 'Deploy 5 EC2 servers.' With IaC, it overrides everything (the 2 EC2 already in operation), and you get 5 EC2 servers.


Finished the Learn to Code 18 Linux CTF. A good refresher on Linux commands. Onwards! Right now, I will focus on breaking and making mistakes fast first before learning optimizing.

<img width="967" height="975" alt="awww" src="https://github.com/user-attachments/assets/8fe153d7-8f77-460e-9d0c-e832355ad436" />


Finished 4 tickets on the AWS cloud!
I have learnt the inner workings of CloudWatch alarms on error rates, synthetic checks that ping external endpoints every minute and alert if they fail.
I have also learnt how to troubleshoot and connect all Networking basics in a 3-tier VPC.
I lastly learnt how to use subnets checkers, terraform and bash scripts automation to ensure correct networks and allow only the proper security groups or singular IPs.

<img width="1032" height="765" alt="image" src="https://github.com/user-attachments/assets/1f9089c6-b916-4135-bcd2-e83fc75dc4e8" />


Now to run an n8n server to test out some automations using Elastic IPs. Woah!


Just came across my first pseudo-optimization!
So, running multiple n8n workflows on a 1GB RAM t3.micro was something I did not know was a strecth on resources. Lesson well learnt as I had to check using docker ps to compare container uptimes and to see I had only ~118MB of free room to begin with! n8n's process choked, crashed, and Docker automatically restarted it (that's what "restart: always" in the compose file does). But since the memory pressure never actually went away, it crashed again. And again. That's the loop I saw in the logs = "Initializing → ready → crashed → Initializing → ready → crashed," over and over and over and over and you get it.
The fix: adding a 2GB swap file. It'll give n8n the headroom to survive memory spikes without crashing. But a swap is a bandage, not a permanent cure.

2 things I hate: wasting money and being unsafe. 
I was unsafe: I got my accessKeys leaked. 
A good lesson on how to remove IAM access and quickly before a hacker drained my credits:
```bash
:~$ aws s3 ls
2026-06-30 15:05:23 nana-iam-test-1782826819
:~$ aws s3 ls s3://nana-iam-test-1782826819 --recursive
2026-06-30 15:05:43         54 test-file.txt
:~$ aws s3api get-bucket-policy --bucket nana-iam-test-1782826819 2>&1
```


An error occurred (NoSuchBucketPolicy) when calling the GetBucketPolicy operation: The bucket policy does not exist
```bash
:~$ aws s3api get-public-access-block --bucket nana-iam-test-1782826819 2>&1
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
:~$ aws s3 rm s3://nana-iam-test-1782826819 --recursive
aws s3 rb s3://nana-iam-test-1782826819
delete: s3://nana-iam-test-1782826819/test-file.txt
remove_bucket: nana-iam-test-1782826819
:~$
```
Lesson learnt!

I am putting off journal api for now to go through Ghana's OneMillionCoders' course on cloud practitioner. This is so helpful, because yes, I will admit, I didn't know what an S3 bucket really was. Now, I know forever. I just finished the Learn To Cloud section of making a bucket, adding IAM users and special accesses and deleting everything.



Just wrapped up the creation of the Python, FastAPI and PostgreSQL capstone project I will be working on going forward: a journal API with full CRUD endpoints, Pydantic input validation, structured logging and an OpenAI SDK AI-powered analyzer. I had also gotten an early start on using some GitHub Actions.
I especially like the AI addition, not just because I love AI, but that we are getting the most requested skills in real time instead of material from years ago. AI integration didn't even exist roughly two years ago, and now we have it in almost everything! I also debugged devcontainer failures, git commit workflows and static type checked with Pyright. All 50 tests passing with flying green colours! Gwyneth Peña-Siguenza, your vast knowledge on what an engineer needs to know tomorrow truly shines through and I hope other fellow members are enjoying the course just as much as me.
I will still need to learn more on Pyright/mypy to make more quality, maintainable code. Also, because my laptop is currently full with university projects, I will be learning the cloud on the cloud. Thank you Github Codespaces! I will redo this whole course until everything becomes second nature.


While learning AWS CloudFormation and Elastic Beanstalk for my n8n automation project, I hit a classic cloud challenge: how do you prevent AWS from automatically deleting your live server and wiping out your data during updates?
Instead of just turning on basic deletion locks, I decided to practice true cloud engineering principles:
1. Decoupled State: Separated the OS from the application data using independent Amazon EBS Volumes.
2. Automated Recovery: Configured the infrastructure so that if Elastic Beanstalk terminates the EC2 instance, a fresh replacement instantly provisions, attaches the persistent data volume, and resumes workflows.
3. Static Networking: Linked an AWS Elastic IP to maintain a permanent gateway.
The result? The compute layer is now completely stateless and disposable, while the n8n data is entirely safe and persistent. Treating servers like cattle, not pets!


```bash
#!/bin/bash
#
# vpc_lab.sh — Automates the public/private VPC lab:
#   VPC -> subnets -> IGW -> route table -> security groups -> VMs -> verify
#   ...and a matching teardown in reverse order.
#
# Usage:
#   ./vpc_lab.sh create     # builds everything
#   ./vpc_lab.sh destroy    # tears everything down (reads state file)
#   ./vpc_lab.sh status     # shows current state file contents
#
# All resource IDs get saved to vpc_lab_state.env as they're created,
# so "destroy" doesn't need you to copy/paste anything back in.

set -euo pipefail

STATE_FILE="./vpc_lab_state.env"
REGION="us-east-1"
VPC_CIDR="10.0.0.0/16"
PUB_CIDR="10.0.1.0/24"
PRIV_CIDR="10.0.2.0/24"
AZ_PUB="us-east-1a"
AZ_PRIV="us-east-1b"
KEY_NAME="alarm-app-key"
INSTANCE_TYPE="t3.micro"

# ---------- helpers ----------

log() { echo -e "\n>>> $1"; }

save_var() {
  # save_var NAME value   -> appends/updates NAME=value in the state file
  local name="$1" value="$2"
  touch "$STATE_FILE"
  # remove old line for this var if present, then append fresh
  grep -v "^${name}=" "$STATE_FILE" > "${STATE_FILE}.tmp" 2>/dev/null || true
  mv "${STATE_FILE}.tmp" "$STATE_FILE" 2>/dev/null || true
  echo "${name}=${value}" >> "$STATE_FILE"
}

load_state() {
  if [ ! -f "$STATE_FILE" ]; then
    echo "No state file found ($STATE_FILE). Nothing to load."
    return 1
  fi
  # shellcheck disable=SC1090
  source "$STATE_FILE"
}

wait_for() {
  # wait_for "description" seconds
  local desc="$1" secs="$2"
  log "Waiting ${secs}s for: ${desc}"
  sleep "$secs"
}

get_my_ip() {
  curl -s ifconfig.me
}

get_latest_ubuntu_ami() {
  aws ec2 describe-images \
    --owners 099720109477 \
    --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
               "Name=state,Values=available" \
    --query "sort_by(Images, &CreationDate)[-1].ImageId" \
    --output text --no-cli-pager
}

# ---------- create ----------

do_create() {
  log "Step 1/9: Creating VPC (${VPC_CIDR})"
  VPC_ID=$(aws ec2 create-vpc --cidr-block "$VPC_CIDR" \
    --query "Vpc.VpcId" --output text --no-cli-pager)
  save_var VPC_ID "$VPC_ID"
  aws ec2 create-tags --resources "$VPC_ID" --tags Key=Name,Value=alarm-app-practice-vpc --no-cli-pager
  echo "VPC created: $VPC_ID"
  wait_for "VPC to register fully" 5

  log "Step 2/9: Creating public + private subnets"
  PUB_SUBNET_ID=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block "$PUB_CIDR" \
    --availability-zone "$AZ_PUB" --query "Subnet.SubnetId" --output text --no-cli-pager)
  save_var PUB_SUBNET_ID "$PUB_SUBNET_ID"
  aws ec2 create-tags --resources "$PUB_SUBNET_ID" --tags Key=Name,Value=practice-public-subnet --no-cli-pager

  PRIV_SUBNET_ID=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block "$PRIV_CIDR" \
    --availability-zone "$AZ_PRIV" --query "Subnet.SubnetId" --output text --no-cli-pager)
  save_var PRIV_SUBNET_ID "$PRIV_SUBNET_ID"
  aws ec2 create-tags --resources "$PRIV_SUBNET_ID" --tags Key=Name,Value=practice-private-subnet --no-cli-pager
  echo "Public subnet:  $PUB_SUBNET_ID"
  echo "Private subnet: $PRIV_SUBNET_ID"
  wait_for "subnets to register" 5

  log "Step 3/9: Creating + attaching Internet Gateway"
  IGW_ID=$(aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" \
    --output text --no-cli-pager)
  save_var IGW_ID "$IGW_ID"
  aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID" --no-cli-pager
  echo "IGW created + attached: $IGW_ID"
  wait_for "IGW attachment to propagate" 5

  log "Step 4/9: Creating public route table + route + association"
  RTB_ID=$(aws ec2 create-route-table --vpc-id "$VPC_ID" \
    --query "RouteTable.RouteTableId" --output text --no-cli-pager)
  save_var RTB_ID "$RTB_ID"
  aws ec2 create-route --route-table-id "$RTB_ID" \
    --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID" --no-cli-pager
  RTB_ASSOC_ID=$(aws ec2 associate-route-table --route-table-id "$RTB_ID" \
    --subnet-id "$PUB_SUBNET_ID" --query "AssociationId" --output text --no-cli-pager)
  save_var RTB_ASSOC_ID "$RTB_ASSOC_ID"
  echo "Route table: $RTB_ID (assoc: $RTB_ASSOC_ID)"

  log "Step 5/9: Creating security groups"
  PUB_SG_ID=$(aws ec2 create-security-group --group-name alarm-app-public-sg \
    --description "Public tier SG" --vpc-id "$VPC_ID" \
    --query "GroupId" --output text --no-cli-pager)
  save_var PUB_SG_ID "$PUB_SG_ID"

  PRIV_SG_ID=$(aws ec2 create-security-group --group-name alarm-app-private-sg \
    --description "Private tier SG" --vpc-id "$VPC_ID" \
    --query "GroupId" --output text --no-cli-pager)
  save_var PRIV_SG_ID "$PRIV_SG_ID"

  MY_IP=$(get_my_ip)
  echo "Detected your public IP: ${MY_IP}"

  aws ec2 authorize-security-group-ingress --group-id "$PUB_SG_ID" \
    --protocol tcp --port 443 --cidr 0.0.0.0/0 --no-cli-pager
  aws ec2 authorize-security-group-ingress --group-id "$PUB_SG_ID" \
    --protocol tcp --port 80 --cidr 0.0.0.0/0 --no-cli-pager
  aws ec2 authorize-security-group-ingress --group-id "$PUB_SG_ID" \
    --protocol tcp --port 22 --cidr "${MY_IP}/32" --no-cli-pager

  aws ec2 authorize-security-group-ingress --group-id "$PRIV_SG_ID" \
    --protocol tcp --port 5432 --source-group "$PUB_SG_ID" --no-cli-pager
  # Private tier also needs SSH from your IP directly if you want to hop in
  # from a bastion later — omitted on purpose to keep it fully private for now.

  echo "Public SG:  $PUB_SG_ID"
  echo "Private SG: $PRIV_SG_ID"

  log "Step 6/9: Creating key pair"
  if [ -f "${KEY_NAME}.pem" ]; then
    echo "Key file ${KEY_NAME}.pem already exists locally — reusing it."
  else
    aws ec2 create-key-pair --key-name "$KEY_NAME" \
      --query "KeyMaterial" --output text --no-cli-pager > "${KEY_NAME}.pem"
    chmod 400 "${KEY_NAME}.pem"
    echo "Key pair saved to ${KEY_NAME}.pem"
  fi

  log "Step 7/9: Finding latest Ubuntu 22.04 AMI"
  AMI_ID=$(get_latest_ubuntu_ami)
  save_var AMI_ID "$AMI_ID"
  echo "Using AMI: $AMI_ID"

  log "Step 8/9: Launching public + private VMs"
  PUB_INSTANCE_ID=$(aws ec2 run-instances \
    --image-id "$AMI_ID" --instance-type "$INSTANCE_TYPE" --key-name "$KEY_NAME" \
    --subnet-id "$PUB_SUBNET_ID" --security-group-ids "$PUB_SG_ID" \
    --associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=practice-public-vm}]" \
    --query "Instances[0].InstanceId" --output text --no-cli-pager)
  save_var PUB_INSTANCE_ID "$PUB_INSTANCE_ID"

  PRIV_INSTANCE_ID=$(aws ec2 run-instances \
    --image-id "$AMI_ID" --instance-type "$INSTANCE_TYPE" --key-name "$KEY_NAME" \
    --subnet-id "$PRIV_SUBNET_ID" --security-group-ids "$PRIV_SG_ID" \
    --no-associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=practice-private-vm}]" \
    --query "Instances[0].InstanceId" --output text --no-cli-pager)
  save_var PRIV_INSTANCE_ID "$PRIV_INSTANCE_ID"

  echo "Public VM instance:  $PUB_INSTANCE_ID"
  echo "Private VM instance: $PRIV_INSTANCE_ID"

  wait_for "instances to reach 'running' state" 30

  log "Step 9/9: Verifying"
  aws ec2 describe-instances --instance-ids "$PUB_INSTANCE_ID" "$PRIV_INSTANCE_ID" \
    --query "Reservations[].Instances[].{Name:Tags[0].Value,State:State.Name,PublicIP:PublicIpAddress,PrivateIP:PrivateIpAddress}" \
    --output table --no-cli-pager

  PUB_IP=$(aws ec2 describe-instances --instance-ids "$PUB_INSTANCE_ID" \
    --query "Reservations[0].Instances[0].PublicIpAddress" --output text --no-cli-pager)

  echo -e "\nAll done. State saved to ${STATE_FILE}."
  echo "Test public access with:"
  echo "  ssh -i ${KEY_NAME}.pem ubuntu@${PUB_IP}"
  echo "(The private VM has no public IP on purpose — that IS the proof it's private.)"
}

# ---------- destroy ----------

do_destroy() {
  if ! load_state; then
    echo "Nothing to destroy — no state file present."
    exit 1
  fi

  log "Step 1/6: Terminating instances"
  aws ec2 terminate-instances --instance-ids "$PUB_INSTANCE_ID" "$PRIV_INSTANCE_ID" --no-cli-pager || true
  wait_for "instances to fully terminate" 30

  log "Step 2/6: Deleting security groups"
  aws ec2 delete-security-group --group-id "$PUB_SG_ID" --no-cli-pager || true
  aws ec2 delete-security-group --group-id "$PRIV_SG_ID" --no-cli-pager || true

  log "Step 3/6: Disassociating + deleting public route table"
  aws ec2 disassociate-route-table --association-id "$RTB_ASSOC_ID" --no-cli-pager || true
  aws ec2 delete-route-table --route-table-id "$RTB_ID" --no-cli-pager || true

  log "Step 4/6: Detaching + deleting Internet Gateway"
  aws ec2 detach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID" --no-cli-pager || true
  aws ec2 delete-internet-gateway --internet-gateway-id "$IGW_ID" --no-cli-pager || true

  log "Step 5/6: Deleting subnets"
  aws ec2 delete-subnet --subnet-id "$PUB_SUBNET_ID" --no-cli-pager || true
  aws ec2 delete-subnet --subnet-id "$PRIV_SUBNET_ID" --no-cli-pager || true
  wait_for "subnet deletions to register" 5

  log "Step 6/6: Deleting VPC"
  aws ec2 delete-vpc --vpc-id "$VPC_ID" --no-cli-pager || true

  log "Verifying VPC is gone"
  if aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --no-cli-pager 2>&1 | grep -q "NotFound"; then
    echo "Confirmed: $VPC_ID no longer exists."
  else
    echo "WARNING: VPC may still exist — check the console manually."
  fi

  echo -e "\nTeardown complete. You can delete ${STATE_FILE} and ${KEY_NAME}.pem manually if you're done for good."
}

do_status() {
  if load_state; then
    echo "Current saved state (${STATE_FILE}):"
    cat "$STATE_FILE"
  fi
}

# ---------- main ----------

case "${1:-}" in
  create)  do_create ;;
  destroy) do_destroy ;;
  status)  do_status ;;
  *)
    echo "Usage: $0 {create|destroy|status}"
    exit 1
    ;;
esac
```


