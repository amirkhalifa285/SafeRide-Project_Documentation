# Team Cloud Training Onboarding (Shared AMI + Branch Isolation)

**Created:** March 7, 2026  
**Purpose:** Let multiple teammates run EC2 training safely in parallel using the same base AMI while each teammate pulls their own git branch.

---

## Scope and Safety Model

This guide assumes:
- Owner account has the golden AMI: `ami-03a3037588b0f34f2`
- Region is `il-central-1`
- Team repo is `roadsense-team/roadsense-v2v`
- Each teammate uses their own AWS account, IAM role, PAT, S3 path, and branch

Isolation rules:
- No one trains from `master` unless explicitly intended
- No shared PATs
- No shared EC2 role across accounts
- No shared S3 output prefix between teammates

---

## Part A - Owner Steps (Share AMI to Teammates)

### A1) Collect teammate AWS account IDs
Ask each teammate for their 12-digit AWS Account ID.

### A2) Share the AMI launch permission
AWS Console path:
1. EC2 -> AMIs
2. Select `ami-03a3037588b0f34f2`
3. Actions -> Edit AMI permissions
4. Add each teammate account ID
5. Save

CLI equivalent (run in owner account):
```bash
aws ec2 modify-image-attribute \
  --region il-central-1 \
  --image-id ami-03a3037588b0f34f2 \
  --launch-permission "Add=[{UserId=111122223333},{UserId=444455556666}]"
```

### A3) Share backing snapshot access (if needed)
If launch fails with snapshot permission errors, share the AMI snapshots too:
1. EC2 -> Snapshots
2. Find snapshots used by the AMI
3. Actions -> Modify permissions
4. Add teammate account IDs

Notes:
- If the AMI/snapshots are encrypted with a customer-managed KMS key, also grant those accounts `kms:Decrypt`/`kms:CreateGrant` on that key.
- If encrypted with `aws/ebs` default key and sharing fails, create a copied AMI backed by a customer-managed KMS key, then share that AMI.

### A4) Send teammates the onboarding package
Send:
- Shared AMI ID
- Region (`il-central-1`)
- Repo URL (`https://github.com/roadsense-team/roadsense-v2v.git`)
- This runbook

---

## Part B - Teammate Steps (One-Time Setup in Their Own AWS Account)

### B1) Copy the shared AMI into own account (recommended)
Console path:
1. EC2 -> AMIs -> Shared with me
2. Select owner AMI
3. Actions -> Copy AMI
4. Destination region: `il-central-1`
5. Name example: `roadsense-training-team-<name>-v1`

Why: training remains stable even if owner later deletes/updates original AMI.

### B2) Create S3 bucket for results
Example bucket: `roadsense-results-<name>`

### B3) Create IAM policy with least privilege
Use this policy (replace bucket name):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::roadsense-results-<name>"]
    },
    {
      "Sid": "ObjectRW",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:AbortMultipartUpload"
      ],
      "Resource": ["arn:aws:s3:::roadsense-results-<name>/*"]
    }
  ]
}
```

### B4) Create EC2 role + instance profile
1. IAM -> Roles -> Create role
2. Trusted entity: AWS service -> EC2
3. Attach the S3 policy above
4. Role name example: `RoadSenseTrainerRole-<name>`

### B5) Create GitHub PAT (per teammate)
- Must be personal (not shared)
- Store securely; do not commit or screenshot it
- Choose token type based on teammate account type:
  - **If teammate is an outside collaborator** on `roadsense-team/roadsense-v2v` (not an org member): use **classic PAT**.  
    Fine-grained PAT will usually not list org repos for outside collaborators.
  - **If teammate is an organization member** of `roadsense-team`: fine-grained PAT is OK.
- For classic PAT with private repo over HTTPS git, include `repo` scope.
- For fine-grained PAT:
  - Resource owner must be `roadsense-team` (not personal account)
  - Repository access: `Only select repositories` -> `roadsense-v2v`
  - Permissions: `Contents: Read-only` is sufficient for pull/fetch
- If org policy requires approval, owner/admin must approve the fine-grained PAT request before it works.

---

## Part C - Per-Run Procedure (No Script Edits, Branch-Specific)

### C1) Launch instance
- Launch from **copied AMI** 
- Type: `c6i.xlarge`
- Region: `il-central-1`
- Attach IAM role: `RoadSenseTrainerRole-<name>`
- 50GB gp3 is enough
- User data: leave empty (manual flow)

### C2) SSH and prepare session
```bash
ssh -i <key>.pem ubuntu@<public-ip>
tmux new -s train
```

### C3) Set run variables
```bash
export AWS_DEFAULT_REGION=il-central-1
export RUN_ID="cloud_<name>_001"
export BRANCH="<friend-branch>"
export GITHUB_PAT="<friend-pat>"
export S3_BUCKET="roadsense-results-<name>"
export WORK_DIR="/home/ubuntu/work"
export DATASET_DIR="ml/scenarios/datasets/dataset_v3/base_real"
export EMULATOR_PARAMS="ml/espnow_emulator/emulator_params_measured.json"
```

### C4) Pull the teammate branch (NOT master)
```bash
cd "$WORK_DIR"
git config --global --add safe.directory "$WORK_DIR"
chmod +x "$WORK_DIR/ml/run_docker.sh"

git remote set-url origin "https://${GITHUB_PAT}@github.com/roadsense-team/roadsense-v2v.git"
git fetch origin
git checkout -B "$BRANCH" "origin/$BRANCH"
git pull origin "$BRANCH"

# Remove PAT from remote URL

git remote set-url origin https://github.com/roadsense-team/roadsense-v2v.git
chmod +x "$WORK_DIR/ml/run_docker.sh"
```

### C5) Rebuild Docker image
```bash
cd "$WORK_DIR/ml"
docker build -t roadsense-ml:latest .
```

### C6) Generate dataset (canonical settings)
```bash
cd "$WORK_DIR"
./ml/run_docker.sh generate \
  --base_dir ml/scenarios/base_real \
  --output_dir "$DATASET_DIR" \
  --seed 42 \
  --train_count 25 \
  --eval_count 10 \
  --peer_drop_prob 0.4 \
  --eval_peer_drop_prob 0.0 \
  --min_peers 1 \
  --emulator_params "$EMULATOR_PARAMS"
```

### C7) Enforce eval n=1..5 deterministic buckets
```bash
./ml/run_docker.sh python3 ml/scripts/fix_eval_peer_counts.py \
  --dataset_dir "$DATASET_DIR" \
  --target_counts 1,1,2,2,3,3,4,4,5,5
```

### C8) Start training
```bash
./ml/run_docker.sh train \
  --dataset_dir "$DATASET_DIR" \
  --emulator_params "$EMULATOR_PARAMS" \
  --total_timesteps 10000000 \
  --run_id "$RUN_ID" \
  --output_dir /work/results \
  --eval_use_deterministic_matrix \
  --eval_matrix_peer_counts "1,2,3,4,5" \
  --eval_matrix_episodes_per_bucket 10
```

### C9) Upload results and shutdown
```bash
aws s3 cp "$WORK_DIR/results" "s3://$S3_BUCKET/$RUN_ID" --recursive --region "$AWS_DEFAULT_REGION"
sudo shutdown -h now
```

---

## Quick Checklist Before Every Run

- [ ] AMI in same region as launch (`il-central-1`)
- [ ] IAM role attached to EC2 instance
- [ ] `aws s3 ls s3://<bucket>/` works from instance
- [ ] Unique `RUN_ID` per teammate/run
- [ ] `BRANCH` is teammate branch (not `master` by default)
- [ ] `--output_dir /work/results` is used (container path)
- [ ] Eval matrix covers n=1,2,3,4,5
- [ ] Teammate PAT removed from git remote after pull

---

## Troubleshooting

### `Unable to locate credentials`
- Role not attached, wrong instance profile, or IMDS issue.
- Verify role from EC2 console, then test:
```bash
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### `fatal: detected dubious ownership in repository`
```bash
git config --global --add safe.directory /home/ubuntu/work
```

### `Permission denied` on `run_docker.sh`
```bash
chmod +x /home/ubuntu/work/ml/run_docker.sh
```

### Repo missing in GitHub PAT repository selector
- This is expected if teammate is an **outside collaborator** and they are creating a **fine-grained PAT**.
- Fix: create a **classic PAT** instead (with `repo` scope for private repo HTTPS git access).
- If using fine-grained PAT, ensure `Resource owner = roadsense-team`; if `roadsense-team` is not selectable, account is likely not an org member or org policy blocks fine-grained tokens.
- If classic PAT access is also blocked by org policy, use one of:
  - convert teammate to org member and use fine-grained PAT with approval, or
  - update org PAT policy to permit the required token type.

### AMI launch fails due to snapshot/KMS permission
- Owner must share snapshot permissions and (if encrypted) KMS key access.
- If using `aws/ebs` default key, owner should copy AMI with a customer-managed key and share that copy.

---

## Branch Policy Recommendation

Use naming convention per teammate:
- `exp/<name>/run010-reward`
- `exp/<name>/run010-stability`
- `exp/<name>/run010-obs`

This keeps experiment ownership clear and avoids accidental pulls from the wrong branch.
