# RemotePC

RemotePC is a fast, lightweight remote-control system that lets you use your **Android device** to control your **Windows PC** over the **local network**.  
It includes a modern UI, real-time mouse input, secure communication, and optional chat capabilities between devices.

This project includes:

- **Windows Desktop Application** (C++ / Qt)
- **Android Client Application** (Kotlin)
- **Real-time mouse tracking & input**
- **Keyboard input support (optional)**
- **Dark/Light mode UI**
- **Minimal latency design**

---

## Features

### Remote Mouse & Input  
Move your mouse using your phone's touchscreen with extremely low latency.

### Android Client  
Clean, modern UI with:
- Touchpad-style mouse controls  
- Tap → left click  
- Two-finger → right click  
- Drag gestures

### Windows Desktop App  
- Built with Qt   
- Displays connection logs  
- Processing of incoming commands  
- System tray controls

### Local Network Security  
All communication remains on your LAN.

### Modern Theme System  
- Auto dark/light mode  
- Smooth animations  
- Custom noise backgrounds  
- Responsive layout
- 
## Downloads

Go to the **Releases** page:

You’ll find:

- **RemotePC-Setup.zip** – Windows
- **RemotePC.apk** – Android

---

## 🛠️ Installation

### Windows
1. Download `RemotePC-Setup.zip`
2. Extract the zip in a location that you want the app to be stored at
3. Launch **RemotePC Server**
4. Ensure your PC and Android device are on the same Wi-Fi / LAN

### Android
1. Download the APK from Releases
2. Install it (you may need to allow unknown sources)
3. Open the app and enter your PC’s IP address

---

## How It Works

### Networking
- The Windows server listens on a known TCP port  
- The Android client sends serialized input commands  
- Messages include:
  - Mouse movement deltas  
  - Click events  
  - Scroll  
  - Keyboard input  

### Security
- All data stays inside your LAN  
- No cloud servers  
- Optional key-based encryption for chat

---

This repository is **not open source**.  
It exists solely to provide official **downloads**, **updates**, and **project information**.
No source code is provided, and redistribution or modification is not permitted.
