建立ubuntu機器t3.medium<br />
開所有port允許anywhere<br />
硬碟35GB gp3，不使用金鑰，使用密碼raspberry<br />
<pre>
bash -c '
set -euo pipefail
trap "echo [ERROR] 發生錯誤，上一個指令失敗於第 $LINENO 行" ERR

export AWS_PAGER=""
export AWS_CLI_PAGER=""
export AWS_DEFAULT_OUTPUT=text

echo "[1/8] 取得 AWS 區域..."
REGION="${AWS_REGION:-${AWS_DEFAULT_REGION:-}}"
if [ -z "$REGION" ]; then
  REGION=$(aws configure get region 2>/dev/null || true)
fi
if [ -z "$REGION" ]; then
  REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep region | cut -d\" -f4 || true)
fi
if [ -z "$REGION" ]; then
  echo "[ERROR] 無法取得區域。" >&2
  exit 1
fi
echo "區域：$REGION"

echo "[2/8] 取得最新 Ubuntu 22.04 AMI..."
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" "Name=state,Values=available" \
  --region "$REGION" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)
echo "使用 AMI：$AMI_ID"

echo "[3/8] 取得 VPC..."
VPC_ID=$(aws ec2 describe-vpcs --region "$REGION" --query "Vpcs[0].VpcId" --output text)
echo "VPC：$VPC_ID"

echo "[4/8] 取得該 VPC 子網..."
SUBNET_ID=$(aws ec2 describe-subnets \
  --region "$REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" \
  --output text)
echo "Subnet：$SUBNET_ID"

echo "[5/8] 建立 Security Group..."
SG_ID=$(aws ec2 create-security-group \
  --region "$REGION" \
  --group-name "open-all-$(date +%s)" \
  --description "open all ports (dangerous)" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)
echo "Security Group ID：$SG_ID"

echo "[6/8] 設定防火牆規則 (全開)..."
aws ec2 authorize-security-group-ingress --region "$REGION" --group-id "$SG_ID" \
  --ip-permissions IpProtocol=-1,IpRanges="[{CidrIp=0.0.0.0/0}]",Ipv6Ranges="[{CidrIpv6=::/0}]" || true
aws ec2 authorize-security-group-egress --region "$REGION" --group-id "$SG_ID" \
  --ip-permissions IpProtocol=-1,IpRanges="[{CidrIp=0.0.0.0/0}]",Ipv6Ranges="[{CidrIpv6=::/0}]" || true

echo "[7/8] 建立 EC2 instance..."
INSTANCE_ID=$(aws ec2 run-instances \
  --region "$REGION" \
  --image-id "$AMI_ID" \
  --instance-type t3.medium \
  --subnet-id "$SUBNET_ID" \
  --associate-public-ip-address \
  --security-group-ids "$SG_ID" \
  --block-device-mappings "[{\"DeviceName\":\"/dev/sda1\",\"Ebs\":{\"VolumeSize\":35,\"VolumeType\":\"gp3\",\"DeleteOnTermination\":true}}]" \
  --user-data "#cloud-config
ssh_pwauth: true
chpasswd:
  list: |
    ubuntu:raspberry
  expire: False
runcmd:
  - sed -i \"s/^#\\?PasswordAuthentication .*/PasswordAuthentication yes/\" /etc/ssh/sshd_config
  - systemctl restart ssh || systemctl restart sshd
" \
  --query "Instances[0].InstanceId" \
  --output text)
echo "Instance ID：$INSTANCE_ID"
  
echo "[8/8] 等待啟動並取得 Public IP..."
aws ec2 wait instance-running --region "$REGION" --instance-ids "$INSTANCE_ID"

IP=$(aws ec2 describe-instances --region "$REGION" --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

echo "---------"
echo "建立完成"
echo "Instance ID: $INSTANCE_ID"
echo "IP: $IP"
echo "使用者名稱: ubuntu"
echo "密碼: raspberry"
echo "SSH 連線指令: ssh ubuntu@$IP"
echo "---------"
'
</pre>

### 預覽圖
<img width="1783" height="543" alt="image" src="https://github.com/user-attachments/assets/d9adbbb4-5817-40db-9392-444a84b41556" />
