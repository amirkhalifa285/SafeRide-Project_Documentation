# Installing Large Python Packages (PyTorch) on Linux

**Context:**
On Linux systems where `/tmp` is mounted as a `tmpfs` (RAM disk), it often has limited size (e.g., 4GB). When installing large packages like PyTorch (~1GB compressed, ~5GB uncompressed), `pip` uses `/tmp` to unpack files before installing them. This can lead to `OSError: [Errno 28] No space left on device`.

**Symptoms:**
- `pip install torch` fails with "No space left on device" even though the main disk has plenty of space.
- `df -h` shows `/tmp` is 100% full during installation.

**Solution:**
Force `pip` to use a temporary directory on the main disk (physical storage) instead of the default `/tmp` RAM disk.

## Command

```bash
# 1. Create a temporary build directory on your main disk
mkdir -p $PROJECT_ROOT/tmp_build

# 2. Run pip with TMPDIR env var and --cache-dir flag
TMPDIR=$PROJECT_ROOT/tmp_build pip install --cache-dir $PROJECT_ROOT/tmp_build -r requirements.txt

# 3. Clean up
rm -rf $PROJECT_ROOT/tmp_build
```

## Example (RoadSense)

```bash
mkdir -p roadsense-v2v/ml/tmp_build
TMPDIR=/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/tmp_build \
roadsense-v2v/ml/venv/bin/pip install \
--cache-dir /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/tmp_build \
-r roadsense-v2v/ml/requirements.txt
rm -rf roadsense-v2v/ml/tmp_build
```

**Tags:** #linux #python #pip #pytorch #troubleshooting

```