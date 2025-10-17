# Running STF

This guide describes how to run STF (Smartphone Test Farm) using different methods: Docker Compose, Docker, or native installation.

- The local URL to access STF is:
  - http://localhost:7100

- You can also access it from other devices on your network using your machine's IP address:
  - http://10.0.0.90:7100

(check `docker-compose.yml`)
- Email: test@google.com
- Name: admin

## Table of Contents

- [Prerequisites](#prerequisites)
- [Method 1: Docker Compose (Recommended)](#method-1-docker-compose-recommended)
- [Method 2: Docker (Manual)](#method-2-docker-manual)
- [Method 3: Native Installation](#method-3-native-installation)
- [Connecting Devices](#connecting-devices)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### All Methods
- **Android devices** with USB debugging enabled
- **USB connection** to your computer
- **ADB installed** (Android Debug Bridge)

### Docker Methods
- [Docker](https://www.docker.com/) installed (20.10+)
- [Docker Compose](https://docs.docker.com/compose/) (for Method 1)
- Devices connected via USB

### Native Installation
- **Node.js** >= 18.20.5 and up to 20.x (versions beyond 20.x not supported)
- **RethinkDB** >= 2.2
- **ADB** properly set up in PATH
- **GraphicsMagick** for screenshot resizing
- **ZeroMQ** libraries
- **Protocol Buffers** libraries
- **CMake** >= 3.9
- **yasm**
- **pkg-config**

---

## Method 1: Docker Compose (Recommended)

This is the easiest way to run STF with all dependencies containerized.

### Step 1: Configure docker-compose.yaml

Edit the `docker-compose.yaml` file in the project root:

```yaml
version: "3"

services:
  rethinkdb:
    container_name: rethinkdb
    image: rethinkdb:2.4.2
    restart: unless-stopped
    volumes:
        - "rethinkdb-data:/data"
    command: "rethinkdb --bind all --cache-size 2048"

  adb:
    container_name: adb
    image: devicefarmer/adb:latest
    restart: unless-stopped
    volumes:
      - "/dev/bus/usb:/dev/bus/usb"
    privileged: true

  stf:
    container_name: stf
    image: devicefarmer/stf
    ports:
      - "7100:7100"
      - "7110:7110"
      - "7400-7500:7400-7500"
    environment:
      - TZ='America/Los_Angeles'
      - RETHINKDB_PORT_28015_TCP=tcp://rethinkdb:28015
      - STF_ADMIN_EMAIL=admin@example.com      # Change this
      - STF_ADMIN_NAME=Administrator            # Change this
    restart: unless-stopped
    command: stf local --adb-host adb --public-ip YOUR_IP --provider-min-port 7400 --provider-max-port 7500

volumes:
  rethinkdb-data: {}
```

**Important:** Replace the following values:
- `STF_ADMIN_EMAIL`: Your email address
- `STF_ADMIN_NAME`: Your name
- `YOUR_IP`: Your machine's IP address (use `hostname -I` on Linux or `ipconfig getifaddr en0` on macOS)

### Step 2: Start Services

```bash
# Start all services in background
docker-compose up -d

# View logs
docker-compose logs -f stf

# Check status
docker-compose ps
```

### Step 3: Access STF

Open your browser and navigate to:
```
http://localhost:7100
```

### Step 4: Stop Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (deletes database)
docker-compose down -v
```

### Docker Compose Notes

- **Data persistence**: RethinkDB data is stored in a Docker volume named `rethinkdb-data`
- **Device access**: The ADB container needs privileged mode to access USB devices
- **Port range**: Ports 7400-7500 are used for device screen streaming
- **Timezone**: Adjust `TZ` environment variable as needed

---

## Method 2: Docker (Manual)

Run STF using Docker containers manually without Docker Compose.

### Step 1: Create a Docker Network

```bash
docker network create stf-network
```

### Step 2: Start RethinkDB

```bash
docker run -d \
  --name rethinkdb \
  --network stf-network \
  -v rethinkdb-data:/data \
  rethinkdb:2.4.2 \
  rethinkdb --bind all --cache-size 2048
```

### Step 3: Start ADB Server

```bash
docker run -d \
  --name adbd \
  --privileged \
  -v /dev/bus/usb:/dev/bus/usb \
  --network stf-network \
  devicefarmer/adb:latest
```

### Step 4: Start STF

```bash
docker run -d \
  --name stf \
  --network stf-network \
  -e "RETHINKDB_PORT_28015_TCP=tcp://rethinkdb:28015" \
  -e "STF_ADMIN_EMAIL=admin@example.com" \
  -e "STF_ADMIN_NAME=Administrator" \
  -p 7100:7100 \
  -p 7110:7110 \
  -p 7400-7500:7400-7500 \
  devicefarmer/stf:latest \
  stf local --adb-host adbd --public-ip YOUR_IP \
  --provider-min-port 7400 --provider-max-port 7500
```

**Replace:**
- `YOUR_IP`: Your machine's IP address
- `STF_ADMIN_EMAIL` and `STF_ADMIN_NAME`: Your credentials

### Step 5: View Logs

```bash
# View STF logs
docker logs -f stf

# View RethinkDB logs
docker logs -f rethinkdb

# View ADB logs
docker logs -f adbd
```

### Step 6: Access STF

Open your browser and navigate to:
```
http://localhost:7100
```

### Step 7: Stop and Clean Up

```bash
# Stop containers
docker stop stf adbd rethinkdb

# Remove containers
docker rm stf adbd rethinkdb

# Remove network
docker network rm stf-network

# Remove volume (optional - deletes database)
docker volume rm rethinkdb-data
```

### Building Your Own Docker Image

If you want to build from source:

```bash
# Build the image
docker build -t my-stf:latest .

# Run with your custom image
docker run -d \
  --name stf \
  --network stf-network \
  -e "RETHINKDB_PORT_28015_TCP=tcp://rethinkdb:28015" \
  -p 7100:7100 \
  -p 7110:7110 \
  -p 7400-7500:7400-7500 \
  my-stf:latest \
  stf local --public-ip YOUR_IP
```

---

## Method 3: Native Installation

Run STF directly on your machine without containers.

### Step 1: Install System Dependencies

#### macOS (via Homebrew)

```bash
brew install rethinkdb graphicsmagick zeromq protobuf yasm pkg-config cmake
```

#### Ubuntu/Debian

```bash
sudo apt-get update
sudo apt-get install -y \
  rethinkdb \
  graphicsmagick \
  libzmq3-dev \
  libprotobuf-dev \
  git \
  yasm \
  cmake \
  pkg-config
```

#### Notes
- **Windows**: Not officially supported. Use Docker instead.
- **ADB**: Ensure Android SDK Platform Tools are installed and `adb` is in your PATH

### Step 2: Install Node.js

Install Node.js 18.20.5 or 20.x (versions beyond 20.x not supported):

```bash
# Using nvm (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 20
nvm use 20

# Verify installation
node --version  # Should show v20.x
npm --version
```

### Step 3: Clone and Install STF

```bash
# Clone the repository
git clone https://github.com/DeviceFarmer/stf.git
cd stf

# Install dependencies (this also builds the frontend)
npm install

# Optionally link the stf command globally
npm link
```

**Note:** `npm install` automatically runs `bower install` and `gulp build` via the prepare script.

### Step 4: Start RethinkDB

Open a new terminal and start RethinkDB:

```bash
# Create a directory for RethinkDB data
mkdir -p ~/stf-rethinkdb
cd ~/stf-rethinkdb

# Start RethinkDB
rethinkdb
```

RethinkDB web interface will be available at: http://localhost:8080

**Important:** Keep this terminal open while using STF.

### Step 5: Configure Built-in Objects (Optional)

Set environment variables before starting STF:

```bash
export STF_ROOT_GROUP_NAME="Common"
export STF_ADMIN_NAME="administrator"
export STF_ADMIN_EMAIL="admin@example.com"
```

### Step 6: Start STF

In a new terminal (with RethinkDB still running):

```bash
cd stf

# Start STF in local mode
stf local

# Or with a specific device (by serial number)
stf local 0123456789ABCDEF

# Or with external access
stf local --public-ip $(hostname -I | awk '{print $1}')
```

**Wait for the build process to complete.** You'll see webpack bundling the frontend assets.

### Step 7: Access STF

Open your browser and navigate to:
```
http://localhost:7100
```

**Default credentials:**
- **Email**: administrator@fakedomain.com (or your `STF_ADMIN_EMAIL`)
- **Name**: administrator (or your `STF_ADMIN_NAME`)

### Step 8: Stop STF

Press `Ctrl+C` in the terminal where STF is running, then stop RethinkDB in its terminal.

### Development Workflow

For development with auto-reloading:

```bash
# Terminal 1: RethinkDB
rethinkdb

# Terminal 2: STF
npm run local

# Terminal 3 (optional): Run tests
npm test
```

### Updating STF

```bash
cd stf
git pull origin master
npm install

# If you encounter issues, clean and reinstall
rm -rf node_modules res/bower_components
npm install
```

---

## Connecting Devices

### Enable USB Debugging on Android Device

1. Go to **Settings** → **About phone**
2. Tap **Build number** 7 times to enable Developer mode
3. Go to **Settings** → **Developer options**
4. Enable **USB debugging**
5. Connect device via USB
6. Accept the authorization prompt on your device

### Verify ADB Connection

```bash
# List connected devices
adb devices

# You should see output like:
# List of devices attached
# 0123456789ABCDEF    device
```

### Checking Device in STF

1. Open STF in your browser (http://localhost:7100)
2. Your device should appear in the device list
3. Click **Use** to start controlling the device

### Multiple Devices

STF automatically detects all connected devices. Simply plug them in and they'll appear in the device list.

### Filtering Devices

To run STF with specific devices only:

```bash
# Native installation
stf local SERIAL1 SERIAL2 SERIAL3

# Docker
docker run ... devicefarmer/stf:latest \
  stf local --adb-host adbd SERIAL1 SERIAL2
```

---

## Troubleshooting

### Device Not Showing Up

**Check ADB:**
```bash
adb devices
```

**Common solutions:**
- Reconnect the USB cable
- Try a different USB port
- Try a different USB cable (some cables are charge-only)
- Revoke USB debugging authorizations on device and try again
- Restart ADB server: `adb kill-server && adb start-server`
- On macOS: Switch between MTP and PTP modes in device USB settings

### Port Already in Use

```bash
# Find process using port 7100
lsof -i :7100

# Kill the process
kill -9 <PID>
```

### RethinkDB Connection Issues

**Native installation:**
- Ensure RethinkDB is running before starting STF
- Check if RethinkDB is accessible at http://localhost:8080
- Try deleting `rethinkdb_data` folder and restarting

**Docker:**
- Check if rethinkdb container is running: `docker ps`
- Check logs: `docker logs rethinkdb`

### Permission Denied on USB Devices (Linux)

```bash
# Add your user to the plugdev group
sudo usermod -aG plugdev $USER

# Create udev rule
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="*", MODE="0666", GROUP="plugdev"' | \
  sudo tee /etc/udev/rules.d/51-android.rules

# Reload udev rules
sudo udevadm control --reload-rules
sudo udevadm trigger

# Log out and log back in for group changes to take effect
```

### STF Web UI Not Loading

- Clear browser cache and reload
- Check browser console for errors
- Ensure all STF processes are running
- Try accessing from `http://127.0.0.1:7100` instead of `localhost`

### Device Stuck on "Preparing..."

- Unplug and replug the device
- Restart STF
- Check `adb devices` to ensure device is authorized
- On device, go to Developer Options and try disabling/enabling USB debugging

### macOS: RethinkDB Takes Long Time to Start

This is a known issue. Fix it by setting the hostname:

```bash
# Check if hostname is unset
scutil --get HostName

# If empty, set it
sudo scutil --set HostName $(hostname)
```

### Build Failures During npm install

**Native installation:**

```bash
# Clean everything and retry
rm -rf node_modules res/bower_components .eslintcache
npm cache clean --force
npm install
```

**Common issues:**
- **Node.js version**: Ensure you're using Node 18.20.5 - 20.x
- **Python**: Some dependencies need Python 3
- **Build tools**: Ensure all build dependencies are installed

### Docker: Cannot Access USB Devices

Ensure the ADB container runs with `--privileged` flag and has access to USB devices:

```bash
docker run --privileged -v /dev/bus/usb:/dev/bus/usb devicefarmer/adb:latest
```

### Performance Issues

- Reduce screen refresh rate in STF settings
- Close unused browser tabs
- Use wired USB connection instead of WiFi ADB
- Ensure sufficient system resources (CPU, RAM)
- On macOS, don't use for production (known ADB reliability issues)

### Getting Help

- Check the [GitHub Issues](https://github.com/DeviceFarmer/stf/issues)
- Review [DEPLOYMENT.md](doc/DEPLOYMENT.md) for production setup
- Check device database at [stf-device-db](https://github.com/devicefarmer/stf-device-db)

---

## Next Steps

- **Production Deployment**: See [DEPLOYMENT.md](doc/DEPLOYMENT.md) for systemd + Docker setup
- **API Documentation**: See [doc/API.md](doc/API.md)
- **Contributing**: See [CONTRIBUTING.md](CONTRIBUTING.md)
- **Testing**: See [TESTING.md](TESTING.md)
