# EC2 Training Guide — Run 010 (For Teammates)

**Last Updated:** March 8, 2026

Each person runs training with a DIFFERENT seed on their own EC2 instance.
After all runs finish, we compare results and pick the best model.

Seeds taken so far:
- Amir: 42
- Person 2: 43
- Person 3: 7

If you need a new seed, pick any number nobody else is using.

---

## Before You Start

You need:
- SSH key: `saferide-key.pem` (ask Amir if you don't have it)
- Your EC2 instance IP address
- A GitHub Personal Access Token (PAT) with repo access

---

## Step-by-Step

### 1. SSH in and start tmux

```bash
ssh -i ~/Downloads/saferide-key.pem ubuntu@<INSTANCE_IP>
tmux new -s train
```

### 2. Pull latest code

```bash
cd /home/ubuntu/work
git pull origin master
chmod +x ml/run_docker.sh
```

If git complains about uncommitted changes:
```bash
git stash
git pull origin master
chmod +x ml/run_docker.sh
```

**If you want to train from YOUR branch instead of master:**
```bash
git pull origin your-branch-name
chmod +x ml/run_docker.sh
```

### 3. Disable auto-shutdown (so instance stays up if something fails)

```bash
sed -i 's|shutdown -h now|echo "DONE — shutdown skipped"|' ml/scripts/cloud/run_training.sh
```

### 4. Patch the script with your seed

Replace `7` below with YOUR seed number:

```bash
sed -i "s/--seed [0-9]*/--seed 7/g" ml/scripts/cloud/run_training.sh
```

### 5. Verify your seed is correct

```bash
grep "seed" ml/scripts/cloud/run_training.sh
```

Every `--seed` line should show YOUR number. If any still show 42 or someone else's seed, re-run the sed from step 4.

### 6. Set your GitHub PAT and run

```bash
export GITHUB_PAT='<YOUR_GITHUB_PAT>'
sudo -E bash ml/scripts/cloud/run_training.sh
```

Training takes about 8 hours.

### 7. Monitor from a second terminal

Open a new terminal window and SSH back in:

```bash
ssh -i ~/Downloads/saferide-key.pem ubuntu@<INSTANCE_IP>
tail -f /var/log/training-run.log
```

### 8. Detach from tmux (training keeps running)

Press `Ctrl+B` then `D` (press Ctrl+B together, release, then press D).

You can close your SSH. Training continues in the background.

To reattach later:
```bash
tmux attach -t train
```

---

## After Training Finishes (~8 hours)

Results upload to S3 automatically. Verify from the instance:

```bash
aws s3 ls s3://saferide-training-results/cloud_prod_010/ --region il-central-1
```

If upload failed, do it manually:

```bash
aws s3 cp /home/ubuntu/work/results s3://saferide-training-results/cloud_prod_010/ --recursive --region il-central-1
```

Then **STOP your EC2 instance** from the AWS Console. Don't leave it running.

---

## Troubleshooting

**"Permission denied" on run_docker.sh:**
```bash
chmod +x /home/ubuntu/work/ml/run_docker.sh
```

**git pull fails with "uncommitted changes":**
```bash
git stash
git pull origin master
chmod +x ml/run_docker.sh
```

**"unrecognized arguments" error during generate:**
Code is outdated. Pull again from step 2.

**Can't see /var/log/training-run.log:**
Training hasn't started writing yet, or the script hasn't been launched. Wait a few seconds and try again.

**Training crashed:**
Check the log, then ask Amir. Don't restart blindly.
