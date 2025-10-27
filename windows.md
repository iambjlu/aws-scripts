建立Windows機器t3.medium<br />
開所有port允許anywhere
自動取得密碼<br />
<pre>
bash -c '
set -euo pipefail
trap "echo [ERROR] 發生錯誤，上一個指令失敗於第 ${LINENO} 行。" ERR

# 參數/設定（可改）
AMI_ID="ami-08d8e0922f4f41d4c"
INSTANCE_TYPE="t3.medium"
VOLUME_SIZE=35
KEYPAIR_PREFIX="temp-key"
CLEAN_KEYPAIR=true    # true = 建完後刪除 AWS 上的 keypair (私鑰本機仍在，腳本最後視情況刪)
PASSWORD_POLL_INTERVAL=15   # 秒
PASSWORD_POLL_TIMEOUT=900   # 秒，最多等待 15 分鐘拿密碼

# aws cli pager 關掉
export AWS_PAGER=""
export AWS_DEFAULT_OUTPUT=text
export AWS_CLI_PAGER=""

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

echo "[2/8] 取得第一個 VPC..."
VPC_ID=$(aws ec2 describe-vpcs --region "$REGION" --query "Vpcs[0].VpcId" --output text)
if [ -z "$VPC_ID" ] || [ "$VPC_ID" = "None" ]; then
  echo "[ERROR] 找不到 VPC。" >&2
  exit 1
fi
echo "VPC: $VPC_ID"

echo "[3/8] 建立 Security Group（全開）..."
SG_NAME="open-all-$(date +%s)"
SG_ID=$(aws ec2 create-security-group \
  --region "$REGION" \
  --group-name "$SG_NAME" \
  --description "open all ports (dangerous)" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)
echo "Security Group ID：$SG_ID"
aws ec2 authorize-security-group-ingress --region "$REGION" --group-id "$SG_ID" \
  --ip-permissions IpProtocol=-1,IpRanges="[{CidrIp=0.0.0.0/0}]",Ipv6Ranges="[{CidrIpv6=::/0}]" || true

aws ec2 authorize-security-group-egress --region "$REGION" --group-id "$SG_ID" \
  --ip-permissions IpProtocol=-1,IpRanges="[{CidrIp=0.0.0.0/0}]",Ipv6Ranges="[{CidrIpv6=::/0}]" || true

echo "[4/8] 建立臨時 KeyPair 用來取得 Windows 密碼..."
TS=$(date +%s)
KEY_NAME="${KEYPAIR_PREFIX}-${TS}"
KEY_FILE="./${KEY_NAME}.pem"

# 建立 key pair 並把 KeyMaterial 存成檔
aws ec2 create-key-pair --region "$REGION" --key-name "$KEY_NAME" --query "KeyMaterial" --output text > "$KEY_FILE"
chmod 400 "$KEY_FILE"
echo "KeyPair 名稱: $KEY_NAME, 私鑰檔: $KEY_FILE"

echo "[5/8] 建立 EC2 instance..."
# 不更動 Windows 密碼，讓 AWS/AMI 自行產生密碼並以 KeyPair 的公鑰加密
INSTANCE_ID=$(aws ec2 run-instances \
  --region "$REGION" \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --security-group-ids "$SG_ID" \
  --block-device-mappings "[{\"DeviceName\":\"/dev/sda1\",\"Ebs\":{\"VolumeSize\":${VOLUME_SIZE},\"VolumeType\":\"gp3\",\"DeleteOnTermination\":true}}]" \
  --key-name "$KEY_NAME" \
  --query "Instances[0].InstanceId" \
  --output text)

if [ -z "$INSTANCE_ID" ] || [ "$INSTANCE_ID" = "None" ]; then
  echo "[ERROR] 建立 instance 失敗，沒有拿到 InstanceId。" >&2
  exit 1
fi
echo "Instance ID：$INSTANCE_ID"

echo "[6/8] 等待 instance 進入 running..."
aws ec2 wait instance-running --region "$REGION" --instance-ids "$INSTANCE_ID"
echo "instance is running."

# 取得 Public IP
IP=$(aws ec2 describe-instances --region "$REGION" --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
if [ -z "$IP" ] || [ "$IP" = "None" ]; then
  echo "[ERROR] 取得 Public IP 失敗。" >&2
  exit 1
fi
echo "Public IP: $IP"

echo "[7/8] 嘗試取得並解密 Windows 密碼 (約需要一兩分鐘)"
START_TS=$(date +%s)
PASSWORD_DATA=""
ELAPSED=0
while [ $ELAPSED -lt $PASSWORD_POLL_TIMEOUT ]; do
  PASSWORD_DATA=$(aws ec2 get-password-data --region "$REGION" --instance-id "$INSTANCE_ID" --query "PasswordData" --output text 2>/dev/null || true)
  if [ -n "$PASSWORD_DATA" ] && [ "$PASSWORD_DATA" != "None" ]; then
    echo "取得加密密碼資料（非空）。"
    break
  fi
  echo "密碼尚未可拿到，等待 ${PASSWORD_POLL_INTERVAL}s 再試（已等 ${ELAPSED}s）..."
  sleep "$PASSWORD_POLL_INTERVAL"
  ELAPSED=$(( $(date +%s) - START_TS ))
done

if [ -z "$PASSWORD_DATA" ] || [ "$PASSWORD_DATA" = "None" ]; then
  echo "[WARN] 超過等待時間但仍無密碼。可能 Windows 還沒完成初始化或 AMI 不支援 get-password-data。"
  echo "你有兩個選項：1) 等較久；2) 使用 SSM 或 cloud-init 在 instance 內設定密碼。"
else
  # 把 base64 解出來為臨時檔
  ENC_FILE="/tmp/${KEY_NAME}-pw.enc"
  echo "$PASSWORD_DATA" | base64 -d > "$ENC_FILE"

  echo "嘗試用私鑰解密（先試 OAEP，失敗再試 PKCS1 v1.5）..."
  DECRYPTED=""
  set +e
  DECRYPTED=$(openssl rsautl -decrypt -oaep -inkey "$KEY_FILE" -in "$ENC_FILE" 2>/dev/null || true)
  if [ -z "$DECRYPTED" ]; then
    DECRYPTED=$(openssl rsautl -decrypt -inkey "$KEY_FILE" -in "$ENC_FILE" 2>/dev/null || true)
  fi
  set -e

  if [ -z "$DECRYPTED" ]; then
    echo "[ERROR] 解密失敗。可能私鑰格式不符或 AMI 使用不同加密方式。檢查 $KEY_FILE 和 AMI 設定。"
  else
    echo "-----------------------------"
    echo "解密 Administrator 密碼："
    echo "$DECRYPTED"
    echo "-----------------------------"
  fi

  # 清除臨時加密檔
  rm -f "$ENC_FILE"
fi

echo "--------"
echo "完成 ✅"
echo " "
echo "Instance ID: $INSTANCE_ID"
echo "IP: $IP"
echo "使用者: Administrator"
echo "密碼: $DECRYPTED"
echo "RDP請使用Windows「遠端桌面連線」，或App Store「Windows App」"
echo " "
'
</pre>

### 預覽圖
<img width="1867" height="816" alt="image" src="https://github.com/user-attachments/assets/c0d5d1e3-5883-4767-8fd1-1d40a248a3bf" />

