# 建立簡易VPC
### 同款配置
<img width="450" alt="image" src="https://github.com/user-attachments/assets/5c163d6f-5771-4d5b-bb2a-7b73b6ed156f" />
<img width="450" alt="image" src="https://github.com/user-attachments/assets/d2a2776b-fbf7-4bc2-979f-2552a12bed40" />


<pre>
bash -c '
set -euo pipefail
trap "echo \"[FATAL] Script stopped at line \$LINENO (exit \$?)\" >&2" ERR

# -------- Configuration --------
REGION=$(aws configure get region || true)
if [ -z "${REGION:-}" ]; then
  REGION=$(aws ec2 describe-availability-zones --query "AvailabilityZones[0].RegionName" --output text 2>/dev/null || echo "ap-northeast-1")
fi

CIDR_VPC="10.0.0.0/16"
PUB1_CIDR="10.0.1.0/24"
PUB2_CIDR="10.0.2.0/24"
PRI1_CIDR="10.0.11.0/24"
PRI2_CIDR="10.0.12.0/24"
TAG_PREFIX="my-vpc-demo"

echo "🏗️  Starting VPC setup in region: $REGION"

# -------- Availability Zones --------
AZS=($(aws ec2 describe-availability-zones --region "$REGION" --query "AvailabilityZones[?State==\`available\`].ZoneName" --output text))
AZ1=${AZS[0]}
AZ2=${AZS[1]}

# -------- Create VPC --------
echo "→ Creating VPC..."
VPC_ID=$(aws ec2 create-vpc --region "$REGION" --cidr-block "$CIDR_VPC" --instance-tenancy default --query "Vpc.VpcId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$VPC_ID" --tags Key=Name,Value="$TAG_PREFIX-vpc"
aws ec2 modify-vpc-attribute --region "$REGION" --vpc-id "$VPC_ID" --enable-dns-hostnames "{\"Value\":true}"
aws ec2 modify-vpc-attribute --region "$REGION" --vpc-id "$VPC_ID" --enable-dns-support "{\"Value\":true}"

# -------- Internet Gateway --------
echo "→ Creating and attaching Internet Gateway..."
IGW_ID=$(aws ec2 create-internet-gateway --region "$REGION" --query "InternetGateway.InternetGatewayId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$IGW_ID" --tags Key=Name,Value="$TAG_PREFIX-igw"
aws ec2 attach-internet-gateway --region "$REGION" --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"

# -------- Public Route Table --------
echo "→ Setting up public route table..."
PUB_RT_ID=$(aws ec2 create-route-table --region "$REGION" --vpc-id "$VPC_ID" --query "RouteTable.RouteTableId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$PUB_RT_ID" --tags Key=Name,Value="$TAG_PREFIX-public-rt"
aws ec2 create-route --region "$REGION" --route-table-id "$PUB_RT_ID" --destination-cidr-block "0.0.0.0/0" --gateway-id "$IGW_ID" >/dev/null

# -------- Public Subnets --------
echo "→ Creating public subnets..."
PUB_SUBNET1_ID=$(aws ec2 create-subnet --region "$REGION" --vpc-id "$VPC_ID" --cidr-block "$PUB1_CIDR" --availability-zone "$AZ1" --query "Subnet.SubnetId" --output text)
PUB_SUBNET2_ID=$(aws ec2 create-subnet --region "$REGION" --vpc-id "$VPC_ID" --cidr-block "$PUB2_CIDR" --availability-zone "$AZ2" --query "Subnet.SubnetId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$PUB_SUBNET1_ID" --tags Key=Name,Value="$TAG_PREFIX-public-a"
aws ec2 create-tags --region "$REGION" --resources "$PUB_SUBNET2_ID" --tags Key=Name,Value="$TAG_PREFIX-public-b"
aws ec2 modify-subnet-attribute --region "$REGION" --subnet-id "$PUB_SUBNET1_ID" --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --region "$REGION" --subnet-id "$PUB_SUBNET2_ID" --map-public-ip-on-launch
aws ec2 associate-route-table --region "$REGION" --subnet-id "$PUB_SUBNET1_ID" --route-table-id "$PUB_RT_ID"
aws ec2 associate-route-table --region "$REGION" --subnet-id "$PUB_SUBNET2_ID" --route-table-id "$PUB_RT_ID"

# -------- Private Subnets --------
echo "→ Creating private subnets..."
PRI_SUBNET1_ID=$(aws ec2 create-subnet --region "$REGION" --vpc-id "$VPC_ID" --cidr-block "$PRI1_CIDR" --availability-zone "$AZ1" --query "Subnet.SubnetId" --output text)
PRI_SUBNET2_ID=$(aws ec2 create-subnet --region "$REGION" --vpc-id "$VPC_ID" --cidr-block "$PRI2_CIDR" --availability-zone "$AZ2" --query "Subnet.SubnetId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$PRI_SUBNET1_ID" --tags Key=Name,Value="$TAG_PREFIX-private-a"
aws ec2 create-tags --region "$REGION" --resources "$PRI_SUBNET2_ID" --tags Key=Name,Value="$TAG_PREFIX-private-b"

# -------- NAT Gateway --------
echo "→ Allocating EIP and creating NAT Gateway (this may take 1–2 min)..."
EIP_ALLOC_ID=$(aws ec2 allocate-address --region "$REGION" --domain vpc --query "AllocationId" --output text)
NAT_GW_ID=$(aws ec2 create-nat-gateway --region "$REGION" --subnet-id "$PUB_SUBNET1_ID" --allocation-id "$EIP_ALLOC_ID" --query "NatGateway.NatGatewayId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$NAT_GW_ID" --tags Key=Name,Value="$TAG_PREFIX-natgw"

# -------- Private Route Table --------
echo "→ Creating private route table..."
PRI_RT_ID=$(aws ec2 create-route-table --region "$REGION" --vpc-id "$VPC_ID" --query "RouteTable.RouteTableId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$PRI_RT_ID" --tags Key=Name,Value="$TAG_PREFIX-private-rt"
aws ec2 associate-route-table --region "$REGION" --subnet-id "$PRI_SUBNET1_ID" --route-table-id "$PRI_RT_ID"
aws ec2 associate-route-table --region "$REGION" --subnet-id "$PRI_SUBNET2_ID" --route-table-id "$PRI_RT_ID"

for i in {1..30}; do
  if aws ec2 create-route --region "$REGION" --route-table-id "$PRI_RT_ID" --destination-cidr-block "0.0.0.0/0" --nat-gateway-id "$NAT_GW_ID" >/dev/null 2>&1; then
    break
  else
    echo "⏳ Waiting for NAT Gateway to become available ($i/30)..."
    sleep 5
  fi
done

# -------- S3 Gateway Endpoint --------
echo "→ Creating S3 Gateway Endpoint..."
S3_EP_ID=$(aws ec2 create-vpc-endpoint --region "$REGION" --vpc-id "$VPC_ID" --service-name "com.amazonaws.$REGION.s3" --vpc-endpoint-type Gateway --route-table-ids "$PRI_RT_ID" --query "VpcEndpoint.VpcEndpointId" --output text)
aws ec2 create-tags --region "$REGION" --resources "$S3_EP_ID" --tags Key=Name,Value="$TAG_PREFIX-s3-endpoint"

# -------- Summary --------
echo
echo "✅ VPC creation complete!"
echo "-----------------------------------"
echo "Region:        $REGION"
echo "VPC ID:        $VPC_ID"
echo "Public Subnets: $PUB_SUBNET1_ID ($AZ1), $PUB_SUBNET2_ID ($AZ2)"
echo "Private Subnets: $PRI_SUBNET1_ID ($AZ1), $PRI_SUBNET2_ID ($AZ2)"
echo "IGW:           $IGW_ID"
echo "NAT Gateway:   $NAT_GW_ID"
echo "Public RT:     $PUB_RT_ID"
echo "Private RT:    $PRI_RT_ID"
echo "S3 Endpoint:   $S3_EP_ID"
echo " "
echo "貼心提示：如欲刪除，請先刪掉NAT Gateway才能刪"
echo "-----------------------------------"
'
</pre>

### 預覽圖
<img width="586" height="745" alt="image" src="https://github.com/user-attachments/assets/712fa532-aba0-4aa5-a52d-ef18766ac04d" />
