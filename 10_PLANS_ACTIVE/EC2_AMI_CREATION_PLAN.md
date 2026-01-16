# EC2 AMI Creation Plan

**Status:** ACTIVE
**Priority:** HIGH - Next Session
**Created:** January 16, 2026

---

## Problem

Currently, each cloud training run requires:
1. Launch fresh EC2 instance
2. Wait for user-data script to install Docker, clone repo, build image
3. ~10-15 min setup time before training starts

## Solution

Create a pre-configured AMI with:
- Docker installed and running
- roadsense-ml:latest image pre-built
- AWS CLI configured
- Git repo cloned

## Steps

- [ ] Launch c6i.xlarge with current user-data script
- [ ] Wait for full setup completion
- [ ] Verify training can start immediately
- [ ] Create AMI from running instance
- [ ] Document AMI ID in this file
- [ ] Test new instance from AMI launches training in <2 min
- [ ] Update cloud training docs with AMI usage

## AMI Details (Fill after creation)

| Field | Value |
|-------|-------|
| AMI ID | TBD |
| Region | il-central-1 |
| Base AMI | Amazon Linux 2023 |
| Created | TBD |

## Benefits

- Faster iteration on training runs
- Reduced setup errors
- Consistent environment across runs
- Lower cost (less idle time during setup)
