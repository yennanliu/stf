# STF System Design Document

**Version:** 1.0
**Last Updated:** 2025-10-17
**Authors:** DeviceFarmer Team

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Architecture](#architecture)
4. [Core Capabilities](#core-capabilities)
5. [Limitations](#limitations)
6. [Integration with Automation Frameworks](#integration-with-automation-frameworks)
7. [Use Cases](#use-cases)
8. [API Design](#api-design)
9. [Deployment Architecture](#deployment-architecture)
10. [Security Considerations](#security-considerations)
11. [Performance & Scalability](#performance--scalability)
12. [Future Enhancements](#future-enhancements)

---

## 1. Executive Summary

**STF (Smartphone Test Farm)** is a web-based device farm management platform that enables teams to remotely access, control, and manage Android devices. It provides the infrastructure layer for device access but is not a test automation framework itself.

### Key Characteristics

- **Type**: Device Farm Management Platform
- **Primary Purpose**: Remote device access and management
- **Architecture**: Distributed microservices
- **Communication**: ZeroMQ + Protocol Buffers
- **Supported Platforms**: Android 2.3.3 (SDK 10) to Android 15 (SDK 35)
- **Deployment**: Docker, Kubernetes, Systemd

### What STF Is

✅ Device farm management system
✅ Remote device control interface
✅ Device allocation and booking platform
✅ ADB proxy and multiplexer
✅ RESTful API for programmatic access

### What STF Is NOT

❌ Test automation framework
❌ CI/CD system
❌ Test script recorder/player
❌ Performance testing tool
❌ Mobile app development IDE

---

## 2. System Overview

### 2.1 Purpose

STF solves the problem of **physical device access at scale**. Instead of requiring developers/testers to have physical access to devices, STF provides:

1. **Centralized device management** - All devices in one location
2. **Remote access** - Control devices from anywhere
3. **Multi-user support** - Share devices across teams
4. **API-driven access** - Integrate with automation tools

### 2.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Users/Clients                        │
│  (Web Browser, REST API Clients, Automation Frameworks)     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ HTTP/WebSocket
                         │
┌────────────────────────▼────────────────────────────────────┐
│                      STF Frontend Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   App    │  │   Auth   │  │   API    │  │ WebSocket│   │
│  │  Server  │  │  Server  │  │  Server  │  │  Server  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ ZeroMQ (PUB/SUB, DEALER/ROUTER)
                         │
┌────────────────────────▼────────────────────────────────────┐
│                    Communication Layer                       │
│  ┌────────────────────────────────────────────────────┐    │
│  │            TriProxy (App/Dev Bridges)              │    │
│  └────────────────────────────────────────────────────┘    │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌────────────┐  ┌────────────┐  ┌────────────┐
│ Processor  │  │ Processor  │  │ Processor  │
│   Unit 1   │  │   Unit 2   │  │   Unit N   │
└─────┬──────┘  └─────┬──────┘  └─────┬──────┘
      │               │               │
      └───────────────┼───────────────┘
                      │
                      │ ZeroMQ
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    Provider Layer                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │Provider 1│  │Provider 2│  │Provider 3│  │Provider N│   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        │ ADB         │ ADB         │ ADB         │ ADB
        │             │             │             │
   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
   │ Device  │   │ Device  │   │ Device  │   │ Device  │
   │    1    │   │    2    │   │    3    │   │    N    │
   └─────────┘   └─────────┘   └─────────┘   └─────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Supporting Services                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │RethinkDB │  │ Storage  │  │  Reaper  │  │  Groups  │   │
│  │          │  │  Plugins │  │          │  │  Engine  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Component Overview

| Component | Purpose | Scalability |
|-----------|---------|-------------|
| **App Unit** | Serves web UI and static assets | Horizontal |
| **Auth Unit** | Handles authentication (OAuth2, LDAP, SAML2) | Horizontal |
| **API Unit** | RESTful API endpoints | Horizontal |
| **WebSocket Unit** | Real-time communication with browser | Horizontal |
| **Processor Unit** | Message routing and device coordination | Horizontal |
| **Provider Unit** | ADB connection and device workers | 1 per host |
| **TriProxy Units** | ZeroMQ message proxies (app/dev bridges) | Limited |
| **Reaper Unit** | Monitors device heartbeats | Singleton |
| **Groups Engine** | Device booking/partitioning logic | Singleton |
| **Storage Units** | APK/image/file storage | Horizontal |
| **RethinkDB** | Device state, users, groups database | Cluster |

---

## 3. Architecture

### 3.1 Microservices Architecture

STF follows a **distributed microservices** pattern where each unit is:

- **Independent** - Can be deployed separately
- **Stateless** (mostly) - Except for database-backed units
- **Scalable** - Most units can scale horizontally
- **Fault-tolerant** - Units can restart without affecting others

### 3.2 Communication Patterns

#### ZeroMQ Patterns Used

1. **PUB/SUB** - Broadcast events (device status, user actions)
2. **DEALER/ROUTER** - Bidirectional request/response
3. **PUSH/PULL** - Work distribution

#### Message Flow Example: Device Control

```
User Action (Browser)
    │
    │ WebSocket
    ▼
WebSocket Unit
    │
    │ ZeroMQ PUSH
    ▼
TriProxy App
    │
    │ ZeroMQ DEALER
    ▼
Processor Unit
    │
    │ ZeroMQ DEALER
    ▼
TriProxy Dev
    │
    │ ZeroMQ PUB
    ▼
Provider Unit
    │
    │ ADB Command
    ▼
Android Device
```

### 3.3 Data Flow

#### Device Registration

```
1. Provider detects device via ADB
2. Provider publishes device info to TriProxy Dev
3. Processor receives device info
4. Processor stores device in RethinkDB
5. Processor broadcasts device availability
6. WebSocket Unit receives broadcast
7. WebSocket sends device to browser
8. User sees device in device list
```

#### Device Control (Touch Input)

```
1. User touches screen in browser
2. Browser sends coordinates via WebSocket
3. WebSocket Unit forwards to Processor
4. Processor routes to correct Provider
5. Provider executes ADB touch command
6. Device processes touch event
7. Screen updates
8. Provider streams screen via minicap
9. Processor forwards screen data
10. WebSocket sends image to browser
```

### 3.4 Technology Stack

#### Backend
- **Runtime**: Node.js 18.20.5 - 20.x
- **Language**: JavaScript (ES5/ES6)
- **Framework**: Express.js
- **Messaging**: ZeroMQ 4.x
- **Serialization**: Protocol Buffers
- **Database**: RethinkDB 2.2+
- **Process Management**: Systemd, Docker

#### Frontend
- **Framework**: AngularJS 1.x
- **Build Tool**: Webpack 5
- **Task Runner**: Gulp
- **Styling**: LESS, SCSS, CSS
- **Real-time**: Socket.io

#### Device Layer
- **ADB**: Android Debug Bridge
- **minicap**: Screen capture (30-40 FPS)
- **minitouch**: Touch input simulation
- **minirev**: Reverse port forwarding
- **STFService**: Android service APK

---

## 4. Core Capabilities

### 4.1 Device Management

#### Device Discovery
- Automatic detection via ADB
- Device identification (serial, model, manufacturer)
- Hardware specs retrieval
- Network configuration

#### Device Status Tracking
- **Present/Absent** - Physical connection status
- **Ready/Busy** - Availability status
- **Using** - Current user
- **Battery Level** - Real-time monitoring
- **Temperature** - Thermal monitoring

#### Device Allocation
- First-come-first-served model
- Manual device reservation
- Group-based partitioning
- Time-based booking

### 4.2 Remote Control Features

#### Display
- Real-time screen mirroring (30-40 FPS)
- Screen rotation support
- Adjustable quality settings
- Screenshot capture

#### Input Methods
- **Touch** - Single and multi-touch
- **Keyboard** - Type from computer keyboard
- **Gestures** - Pinch, zoom, rotate (with Alt key)
- **Hardware Keys** - Back, Home, Menu, Power

#### Device Interaction
- App installation (drag & drop APK)
- App launching
- File upload/download
- Shell command execution
- Device logs viewing
- Browser opening

### 4.3 Developer Tools

#### ADB Connectivity
```bash
# Connect to remote device
adb connect <stf-device-ip>:<port>

# Run any ADB command
adb -s <serial> shell
adb -s <serial> install app.apk
adb -s <serial> logcat
```

#### Remote Debugging
- Chrome DevTools integration
- Android Studio support
- JDWP debugging
- Network traffic inspection

### 4.4 Team Collaboration

#### User Management
- User registration and authentication
- Role-based access control (User/Admin)
- Personal access tokens for API
- Email-based user identification

#### Group Management
- Device groups/pools
- User groups/teams
- Time-based device allocation
- Group quotas and limits

#### Booking System
- Reserve devices in advance
- Recurring bookings
- Auto-release after timeout
- Email notifications

### 4.5 Storage & Assets

#### Storage Plugins
- **APK Plugin** - Parse APK metadata, manifest
- **Image Plugin** - Resize screenshots, device images
- **Temporary Storage** - In-memory + filesystem
- **S3 Storage** - AWS S3 integration

---

## 5. Limitations

### 5.1 What STF Cannot Do

#### Not a Test Automation Framework
❌ No built-in test scripting language
❌ No test recording/playback
❌ No parallel test execution
❌ No test reporting/analytics
❌ No test case management

**STF provides device access, not test automation.**

#### Not a Performance Testing Tool
❌ No load generation
❌ No performance metrics collection
❌ No network throttling
❌ No CPU/memory profiling

#### Not a Mobile Development IDE
❌ No code editor
❌ No build system
❌ No debugger (except remote ADB)
❌ No emulator management

### 5.2 Platform Limitations

#### Android Only
- No iOS support currently
- Android 2.3.3+ required
- Some features require Android 5.0+

#### USB Connection Required
- Devices must be physically connected via USB
- WiFi ADB possible but unreliable
- USB hubs can cause issues

#### Single Provider Per Host
- Only one provider unit per physical machine
- Providers compete for device access
- Cannot run multiple providers on same host

#### Singleton Units
- **Reaper** - Only one instance allowed
- **Groups Engine** - Only one instance allowed
- **Storage Temp** - Only one instance allowed

---

## 6. Integration with Automation Frameworks

### 6.1 Architecture Pattern: STF + Automation

```
┌────────────────────────────────────────────────────────────┐
│                    Test Orchestration Layer                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Jenkins    │  │   GitHub     │  │   Custom     │    │
│  │   (CI/CD)    │  │   Actions    │  │   Scripts    │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
└─────────┼──────────────────┼──────────────────┼────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                   Test Framework Layer                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Appium  │  │ Robot FW │  │ Espresso │  │  pytest  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        └─────────────┼─────────────┼─────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                      STF Platform                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │  STF API   │  │  ADB Proxy │  │   Device   │           │
│  │            │  │            │  │   Pool     │           │
│  └────┬───────┘  └────┬───────┘  └────┬───────┘           │
└───────┼───────────────┼───────────────┼─────────────────────┘
        │               │               │
        └───────────────┼───────────────┘
                        │
┌───────────────────────▼─────────────────────────────────────┐
│                    Physical Devices                          │
│  [Device 1] [Device 2] [Device 3] ... [Device N]           │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Integration Approach 1: STF API + Appium

#### Workflow

```python
# Step 1: Get available devices from STF
import requests

STF_URL = "http://localhost:7100"
TOKEN = "your_access_token"

headers = {"Authorization": f"Bearer {TOKEN}"}

# Get available devices
devices = requests.get(f"{STF_URL}/api/v1/devices", headers=headers).json()
available = [d for d in devices['devices'] if d['present'] and not d['using']]

# Step 2: Reserve a device
device_serial = available[0]['serial']
requests.post(
    f"{STF_URL}/api/v1/user/devices",
    headers=headers,
    json={"serial": device_serial}
)

# Step 3: Get remote connect URL
remote = requests.post(
    f"{STF_URL}/api/v1/user/devices/{device_serial}/remoteConnect",
    headers=headers
).json()

adb_url = remote['remoteConnectUrl']  # e.g., "127.0.0.1:7404"

# Step 4: Connect via ADB
import subprocess
subprocess.run(["adb", "connect", adb_url])

# Step 5: Run Appium test
from appium import webdriver

caps = {
    "platformName": "Android",
    "udid": device_serial,
    "automationName": "UiAutomator2",
    "appPackage": "com.google.android.youtube",
    "appActivity": ".HomeActivity"
}

driver = webdriver.Remote("http://localhost:4723/wd/hub", caps)

# Your test code
driver.find_element_by_id("search_button").click()
driver.find_element_by_id("search_edit_text").send_keys("test video")
driver.press_keycode(66)  # Enter
time.sleep(2)
driver.find_element_by_xpath("//android.widget.TextView[1]").click()

# Step 6: Release device
requests.delete(
    f"{STF_URL}/api/v1/user/devices/{device_serial}",
    headers=headers
)
```

### 6.3 Integration Approach 2: Selenium Grid + STF

```
┌────────────────────────────────────────────────────────────┐
│                    Selenium Grid Hub                        │
│                   (Test Distribution)                       │
└────────────────────────┬───────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Appium    │  │  Appium    │  │  Appium    │
│  Node 1    │  │  Node 2    │  │  Node N    │
└─────┬──────┘  └─────┬──────┘  └─────┬──────┘
      │               │               │
      │ ADB           │ ADB           │ ADB
      │               │               │
┌─────▼───────────────▼───────────────▼─────────┐
│              STF Device Pool                   │
│  [Device 1] [Device 2] ... [Device N]        │
└───────────────────────────────────────────────┘
```

#### Configuration

```yaml
# docker-compose.yml for Grid + STF integration
version: '3'

services:
  selenium-hub:
    image: selenium/hub:latest
    ports:
      - "4444:4444"

  appium-node-1:
    image: appium/appium:latest
    depends_on:
      - selenium-hub
    environment:
      - SELENIUM_HOST=selenium-hub
      - STF_URL=http://stf:7100
      - STF_TOKEN=${STF_TOKEN}
    volumes:
      - /dev/bus/usb:/dev/bus/usb
    privileged: true

  stf:
    image: devicefarmer/stf:latest
    # ... STF configuration
```

### 6.4 Integration Approach 3: CI/CD Pipeline

#### GitHub Actions Example

```yaml
# .github/workflows/mobile-tests.yml
name: Mobile Test Suite

on: [push, pull_request]

jobs:
  android-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          pip install appium-python-client requests pytest

      - name: Get device from STF
        env:
          STF_URL: ${{ secrets.STF_URL }}
          STF_TOKEN: ${{ secrets.STF_TOKEN }}
        run: |
          python scripts/get_stf_device.py

      - name: Run tests
        run: |
          pytest tests/mobile/ --device=$DEVICE_SERIAL

      - name: Release device
        if: always()
        run: |
          python scripts/release_stf_device.py
```

### 6.5 Example Use Case: Multi-Device YouTube Automation

#### Scenario
Run the same YouTube test on 5 devices simultaneously:
1. Open YouTube app
2. Search for "test video"
3. Play first result
4. Verify video plays for 10 seconds

#### Implementation

```python
# test_youtube_parallel.py
import concurrent.futures
from stf_client import STFClient
from appium_test import YouTubeTest

def run_test_on_device(device_serial):
    """Run YouTube test on a single device"""
    stf = STFClient(url="http://localhost:7100", token="YOUR_TOKEN")

    try:
        # 1. Reserve device
        stf.add_device(device_serial)

        # 2. Connect via ADB
        adb_url = stf.remote_connect(device_serial)

        # 3. Run test
        test = YouTubeTest(device_serial, adb_url)
        result = test.run()

        return {"device": device_serial, "result": result, "status": "PASS"}

    except Exception as e:
        return {"device": device_serial, "error": str(e), "status": "FAIL"}

    finally:
        # 4. Release device
        stf.remove_device(device_serial)

def main():
    # Get 5 available devices
    stf = STFClient(url="http://localhost:7100", token="YOUR_TOKEN")
    devices = stf.get_available_devices(count=5)

    # Run tests in parallel
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        futures = [executor.submit(run_test_on_device, d['serial'])
                   for d in devices]

        results = [f.result() for f in concurrent.futures.as_completed(futures)]

    # Print results
    for result in results:
        print(f"Device {result['device']}: {result['status']}")

if __name__ == "__main__":
    main()
```

#### Output
```
Device ABC123: PASS
Device DEF456: PASS
Device GHI789: FAIL (App crashed)
Device JKL012: PASS
Device MNO345: PASS

Summary: 4/5 tests passed
```

---

## 7. Use Cases

### 7.1 Manual Testing
- **Scenario**: QA team needs to test app on 20 different devices
- **Solution**: Use STF web UI to control devices remotely
- **Benefit**: No need to physically access devices

### 7.2 Automated Testing
- **Scenario**: Run regression tests on every commit
- **Solution**: CI/CD pipeline reserves STF devices, runs Appium tests
- **Benefit**: Parallel execution on real devices

### 7.3 Device Lab Management
- **Scenario**: Company has 100+ devices across offices
- **Solution**: STF centralizes all devices with booking system
- **Benefit**: Efficient device utilization

### 7.4 Remote Development
- **Scenario**: Developer needs to debug on specific device
- **Solution**: Use STF + Android Studio remote debugging
- **Benefit**: Debug production issues without device shipping

### 7.5 Demo & Training
- **Scenario**: Sales team needs to demo app on various devices
- **Solution**: Use STF to remotely control devices during presentations
- **Benefit**: No device setup, works from anywhere

---

## 8. API Design

### 8.1 Authentication

```bash
# Generate access token
curl -X POST http://localhost:7100/auth/api/v1/tokens \
  -H "Content-Type: application/json" \
  -d '{"name": "CI Token", "scope": "user"}'

# Response
{
  "success": true,
  "token": "YOUR_ACCESS_TOKEN_HERE"
}
```

### 8.2 Device Management API

#### List Devices
```bash
GET /api/v1/devices
Authorization: Bearer YOUR_TOKEN

Response:
{
  "success": true,
  "devices": [
    {
      "serial": "ABC123",
      "present": true,
      "ready": true,
      "using": false,
      "owner": null,
      "model": "Pixel 6",
      "manufacturer": "Google",
      "sdk": "33",
      "abi": "arm64-v8a",
      "battery": {
        "level": 85,
        "status": "Charging"
      }
    }
  ]
}
```

#### Reserve Device
```bash
POST /api/v1/user/devices
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json

{
  "serial": "ABC123",
  "timeout": 900000
}

Response:
{
  "success": true,
  "description": "Device successfully added"
}
```

#### Get Remote Connect URL
```bash
POST /api/v1/user/devices/{serial}/remoteConnect
Authorization: Bearer YOUR_TOKEN

Response:
{
  "success": true,
  "remoteConnectUrl": "127.0.0.1:7404",
  "remoteConnectUrlSecure": "https://stf.example.com:7404"
}
```

#### Release Device
```bash
DELETE /api/v1/user/devices/{serial}
Authorization: Bearer YOUR_TOKEN

Response:
{
  "success": true,
  "description": "Device successfully removed"
}
```

### 8.3 User Management API

#### List Users (Admin only)
```bash
GET /api/v1/users
Authorization: Bearer ADMIN_TOKEN

Response:
{
  "success": true,
  "users": [
    {
      "email": "user@example.com",
      "name": "John Doe",
      "group": "Common",
      "privilege": "user",
      "devices": 2
    }
  ]
}
```

### 8.4 Group Management API

#### Create Group
```bash
POST /api/v1/groups
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json

{
  "name": "iOS Team",
  "devices": ["ABC123", "DEF456"],
  "users": ["user1@example.com", "user2@example.com"],
  "duration": 86400000,
  "repetitions": 0
}
```

---

## 9. Deployment Architecture

### 9.1 Single-Host Development

```
┌─────────────────────────────────────────────────┐
│            Development Machine (localhost)       │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │     Docker Compose / stf local           │  │
│  │                                          │  │
│  │  [RethinkDB] [ADB] [STF]                │  │
│  └──────────────────────────────────────────┘  │
│                     │                           │
│                     │ USB                       │
│              ┌──────▼──────┐                   │
│              │  Device 1-5 │                   │
│              └─────────────┘                   │
└─────────────────────────────────────────────────┘

Access: http://localhost:7100
```

### 9.2 Small Team (< 50 devices)

```
┌─────────────────────────────────────────────────┐
│              Application Server                  │
│  [App] [Auth] [API] [WebSocket] [Processor]    │
│  [RethinkDB] [Storage] [Reaper] [Groups]       │
└────────────────────┬────────────────────────────┘
                     │
                     │ Network (ZeroMQ)
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼──────┐  ┌────▼──────┐  ┌────▼──────┐
│ Provider  │  │ Provider  │  │ Provider  │
│  Host 1   │  │  Host 2   │  │  Host 3   │
│           │  │           │  │           │
│ Device    │  │ Device    │  │ Device    │
│  1-20     │  │  21-35    │  │  36-50    │
└───────────┘  └───────────┘  └───────────┘

Access: https://stf.company.com
```

### 9.3 Enterprise (> 50 devices)

```
┌─────────────────────────────────────────────────┐
│           Load Balancer (Nginx/HAProxy)         │
└────────────────────┬────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼──────┐  ┌────▼──────┐  ┌────▼──────┐
│ App       │  │ App       │  │ App       │
│ Server 1  │  │ Server 2  │  │ Server N  │
└───────────┘  └───────────┘  └───────────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│        RethinkDB Cluster (3+ nodes)             │
└────────────────────┬────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼──────┐  ┌────▼──────┐  ┌────▼──────┐
│Processor  │  │Processor  │  │Processor  │
│   Pool    │  │   Pool    │  │   Pool    │
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │              │              │
      │              │              │
┌─────▼──────────────▼──────────────▼─────────┐
│         Provider Hosts (10+ hosts)           │
│  [Host1:20] [Host2:20] ... [HostN:20]      │
│                                              │
│         Total: 200+ devices                  │
└──────────────────────────────────────────────┘
```

### 9.4 Kubernetes Deployment

```yaml
# stf-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: stf

---
# stf-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stf-app
  namespace: stf
spec:
  replicas: 3
  selector:
    matchLabels:
      app: stf-app
  template:
    metadata:
      labels:
        app: stf-app
    spec:
      containers:
      - name: stf-app
        image: devicefarmer/stf:latest
        command: ["stf", "app"]
        args:
          - "--port=3000"
          - "--auth-url=https://stf.example.com/auth/mock/"
          - "--websocket-url=wss://stf.example.com/"
        ports:
        - containerPort: 3000
        env:
        - name: RETHINKDB_PORT_28015_TCP
          value: "tcp://rethinkdb:28015"
        - name: SECRET
          valueFrom:
            secretKeyRef:
              name: stf-secret
              key: session-secret

---
# stf-provider-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: stf-provider
  namespace: stf
spec:
  selector:
    matchLabels:
      app: stf-provider
  template:
    metadata:
      labels:
        app: stf-provider
    spec:
      hostNetwork: true
      containers:
      - name: stf-provider
        image: devicefarmer/stf:latest
        command: ["stf", "provider"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: usb
          mountPath: /dev/bus/usb
      volumes:
      - name: usb
        hostPath:
          path: /dev/bus/usb
```

---

## 10. Security Considerations

### 10.1 Current Security Posture

⚠️ **Important**: STF was originally designed as an internal tool and makes assumptions about user trustworthiness.

#### Known Security Limitations

1. **No encryption between processes** - ZeroMQ communication is unencrypted
2. **No device data reset** - User data persists between uses
3. **Privileged access** - Users can execute arbitrary ADB commands
4. **No audit logging** - Limited tracking of user actions
5. **Session management** - Basic cookie-based sessions

### 10.2 Recommended Security Measures

#### Network Level
- Deploy behind VPN or private network
- Use HTTPS for web access (via reverse proxy)
- Firewall ZeroMQ ports (7000-8000 range)
- Isolate device network from corporate network

#### Application Level
- Enable authentication (OAuth2, LDAP, SAML2)
- Use strong session secrets
- Generate unique access tokens per user
- Implement rate limiting on API
- Regular security updates

#### Device Level
- Use test accounts only (no production data)
- Factory reset devices periodically
- Use test devices only
- Disable unnecessary device features
- Monitor device file systems

#### Operational Level
```bash
# Example: Automated device cleanup script
#!/bin/bash
for device in $(adb devices | grep device | awk '{print $1}'); do
  adb -s $device shell pm clear com.android.browser
  adb -s $device shell pm clear com.google.android.youtube
  adb -s $device shell rm -rf /sdcard/Download/*
  adb -s $device shell rm -rf /sdcard/DCIM/*
done
```

---

## 11. Performance & Scalability

### 11.1 Performance Characteristics

#### Screen Streaming
- **FPS**: 30-40 (depends on device and Android version)
- **Latency**: 100-300ms (local network)
- **Bandwidth**: ~1-5 Mbps per device
- **Encoding**: JPEG (via minicap)

#### Device Capacity
| Deployment Size | Devices | Hosts | Database | Estimated Users |
|----------------|---------|-------|----------|-----------------|
| Small | 1-20 | 1 | Single | 5-10 |
| Medium | 20-100 | 3-10 | Single/Cluster | 10-50 |
| Large | 100-500 | 10-50 | Cluster | 50-200 |
| Enterprise | 500+ | 50+ | Cluster | 200+ |

#### Bottlenecks
1. **USB bandwidth** - Limited by USB hub quality
2. **Network bandwidth** - Multiple streams add up
3. **RethinkDB** - Database queries for device state
4. **Provider CPU** - minicap encoding per device

### 11.2 Scaling Strategies

#### Horizontal Scaling
```
Component         | Scaling Strategy
-----------------|----------------------------------
App Unit         | Add more instances behind LB
API Unit         | Add more instances behind LB
WebSocket Unit   | Add more instances behind LB
Processor Unit   | Add more instances (up to ~10)
Provider Unit    | Add more hosts with devices
RethinkDB        | Add more nodes to cluster
```

#### Vertical Scaling
- **CPU**: Provider hosts benefit from more cores
- **Memory**: RethinkDB cache size
- **Network**: 1 Gbps+ recommended for large deployments
- **Storage**: SSD for RethinkDB data

#### Optimization Tips
1. Use wired network (not WiFi) for providers
2. Quality USB hubs (avoid cheap hubs)
3. Reduce screen resolution for faster streaming
4. Use local storage (not network) for temp files
5. Regular RethinkDB maintenance
6. Monitor and restart stuck devices

### 11.3 Monitoring

#### Key Metrics
```
- Device availability (%)
- Device utilization (%)
- Average wait time for device
- Screen streaming FPS
- API response time
- WebSocket latency
- Database query time
- Provider CPU/memory usage
```

#### Monitoring Stack
```
Prometheus + Grafana
    │
    ├─> STF API metrics
    ├─> Device status metrics
    ├─> RethinkDB metrics
    └─> System metrics (CPU, mem, network)
```

---

## 12. Future Enhancements

### 12.1 Roadmap Items

#### Short Term
- ✅ VNC support improvements
- ⏳ Automated device restarts
- ⏳ Better device data cleanup
- ⏳ Performance improvements

#### Long Term
- 🔮 iOS device support
- 🔮 Device farm analytics
- 🔮 Built-in test recorder
- 🔮 Cloud device support
- 🔮 Enhanced security features

### 12.2 Community Contributions

STF is open source and welcomes contributions:
- GitHub: https://github.com/DeviceFarmer/stf
- Issues: https://github.com/DeviceFarmer/stf/issues
- Documentation: https://github.com/DeviceFarmer/stf/tree/master/doc

---

## Appendix A: Comparison with Alternatives

| Feature | STF | Appium | Selenium Grid | AWS Device Farm | Firebase Test Lab |
|---------|-----|--------|---------------|-----------------|-------------------|
| **Type** | Device Farm | Test Framework | Test Distribution | Cloud Device Farm | Cloud Testing |
| **Devices** | Your devices | N/A | N/A | Cloud devices | Cloud devices |
| **Real-time control** | ✅ Yes | ❌ No | ❌ No | ✅ Yes | ❌ No |
| **Test automation** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Self-hosted** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| **Cost** | Free | Free | Free | Pay per minute | Pay per test |
| **iOS support** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |

**STF is best for**: Organizations with physical devices wanting remote access and management.

**Use Appium when**: You need to write and run automated tests.

**Use Selenium Grid when**: You need to distribute tests across multiple nodes.

**Use Cloud Device Farm when**: You don't want to manage physical devices.

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **ADB** | Android Debug Bridge - Command line tool for device communication |
| **minicap** | STF's screen capture tool for Android |
| **minitouch** | STF's touch input injection tool |
| **Provider** | STF unit that manages physical device connections |
| **Processor** | STF unit that routes messages between app and devices |
| **TriProxy** | ZeroMQ proxy for app-side and device-side communication |
| **Reaper** | STF unit that monitors device heartbeats |
| **Groups Engine** | STF unit that manages device booking/partitioning |
| **UDID** | Unique Device Identifier (Android serial number) |
| **ZeroMQ** | High-performance asynchronous messaging library |
| **Protocol Buffers** | Google's serialization format |
| **RethinkDB** | Real-time NoSQL database |

---

## Appendix C: Quick Reference

### Common Commands

```bash
# Start STF
docker-compose up -d
stf local

# Check devices
adb devices
docker exec adb adb devices

# View logs
docker-compose logs -f stf
journalctl -u stf-provider -f

# Stop STF
docker-compose down
systemctl stop stf-*

# Database backup
rethinkdb dump -c rethinkdb:28015

# Get device info via API
curl -H "Authorization: Bearer TOKEN" \
  http://localhost:7100/api/v1/devices
```

### Port Reference

| Port | Service |
|------|---------|
| 7100 | Main HTTP (web UI) |
| 7110 | WebSocket |
| 7102 | Storage temp |
| 7103 | Storage image plugin |
| 7104 | Storage APK plugin |
| 7105 | App server |
| 7106 | API server |
| 7120 | Auth server |
| 7400-7500 | Device screen streaming |
| 28015 | RethinkDB |
| 8080 | RethinkDB web UI |

---

**Document Version**: 1.0
**Last Updated**: 2025-10-17
**Maintained By**: DeviceFarmer Community
