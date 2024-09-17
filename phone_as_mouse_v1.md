# Smartphone as Mouse App - Building Instructions

## Table of Contents

1. [Introduction](#introduction)
2. [Features](#features)
3. [Technology Stack](#technology-stack)
4. [Prerequisites](#prerequisites)
5. [Project Structure](#project-structure)
6. [Setup Instructions](#setup-instructions)
    - [1. Clone the Repository](#1-clone-the-repository)
    - [2. Backend (PC Server) Setup](#2-backend-pc-server-setup)
    - [3. Mobile Application Setup](#3-mobile-application-setup)
7. [Implementation Details](#implementation-details)
    - [1. Backend Server](#1-backend-server)
    - [2. Mobile Application](#2-mobile-application)
8. [Deployment](#deployment)
    - [1. Deploying the Backend Server](#1-deploying-the-backend-server)
    - [2. Deploying the Mobile Application](#2-deploying-the-mobile-application)
9. [Testing](#testing)
10. [Troubleshooting](#troubleshooting)
11. [Conclusion](#conclusion)

---

## Introduction

The **Smartphone as Mouse App** allows users to control their PC's mouse movements and actions using their smartphone. This cross-platform application enhances user convenience by leveraging mobile device capabilities to interact seamlessly with desktop environments.

## Features

- **Real-time Mouse Control:** Move the mouse cursor and perform click actions directly from the smartphone.
- **Cross-Platform Support:** Compatible with Android, iOS, and Windows/Linux/MacOS PCs.
- **Wireless Connectivity:** Connect via Wi-Fi for a stable and fast connection.
- **Customizable Gestures:** Configure gestures for different mouse actions.
- **Secure Connection:** Ensures encrypted communication between devices.

## Technology Stack

- **Frontend (Mobile App):**
  - Framework: React Native
  - Language: TypeScript
  - Libraries: Socket.IO-client, Gesture Handler

- **Backend (PC Server):**
  - Runtime: Node.js
  - Language: TypeScript
  - Libraries: Socket.IO, RobotJS

- **Networking:**
  - Protocol: WebSockets

## Prerequisites

- **For Developers:**
  - Node.js and npm installed
  - React Native CLI installed
  - Android Studio and Xcode (for mobile development)
  - Git installed

- **For Users:**
  - Smartphone with Android or iOS
  - PC with Windows, Linux, or MacOS
  - Both devices connected to the same Wi-Fi network

## Project Structure

```plaintext
smartphone-mouse-app/
├── backend/
│   ├── src/
│   │   ├── index.ts
│   │   └── mouseController.ts
│   ├── package.json
│   └── tsconfig.json
├── mobile/
│   ├── src/
│   │   ├── components/
│   │   │   ├── MouseArea.tsx
│   │   ├── App.tsx
│   │   └── socket.ts
│   ├── package.json
│   └── tsconfig.json
└── README.md
```

## Setup Instructions

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/smartphone-mouse-app.git
cd smartphone-mouse-app
```

### 2. Backend (PC Server) Setup

#### Navigate to Backend Directory

```bash
cd backend
```

#### Install Dependencies

```bash
npm install
```

#### Configure Environment Variables

Create a `.env` file in the `backend` directory:

```bash
touch .env
```

Add the following content:

```env
PORT=3000
```

#### Build and Run the Backend Server

```bash
npm run build
npm start
```

### 3. Mobile Application Setup

#### Navigate to Mobile Directory

```bash
cd ../mobile
```

#### Install Dependencies

```bash
npm install
```

#### Configure Socket Connection

Update the server address in `src/socket.ts`:

```typescript:mobile/src/socket.ts
import io from 'socket.io-client';

const SERVER_IP = '192.168.1.100'; // Replace with your PC's IP address
const SERVER_PORT = 3000;

const socket = io(`http://${SERVER_IP}:${SERVER_PORT}`);

export default socket;
```

#### Run the Mobile Application

```bash
npm run ios
# or
npm run android
```

---

## Implementation Details

### 1. Backend Server

The backend server handles incoming connections from the mobile app and translates them into mouse movements and clicks on the PC.

#### Setting Up the Server

```typescript:backend/src/index.ts
import express from 'express';
import http from 'http';
import { Server } from 'socket.io';
import { moveMouse, clickMouse } from './mouseController';

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
    cors: {
        origin: '*',
    },
});

io.on('connection', (socket) => {
    console.log('A user connected:', socket.id);

    socket.on('move', (data) => {
        moveMouse(data.x, data.y);
    });

    socket.on('click', (data) => {
        clickMouse(data.button);
    });

    socket.on('disconnect', () => {
        console.log('User disconnected:', socket.id);
    });
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

#### Mouse Controller

```typescript:backend/src/mouseController.ts
import robot from 'robotjs';

export function moveMouse(x: number, y: number) {
    robot.moveMouse(x, y);
}

export function clickMouse(button: 'left' | 'right' | 'middle') {
    robot.mouseClick(button);
}
```

### 2. Mobile Application

The mobile app captures touch gestures and translates them into mouse movement and click events sent to the backend server.

#### Socket Configuration

```typescript:mobile/src/socket.ts
import io from 'socket.io-client';

const SERVER_IP = '192.168.1.100'; // Replace with your PC's IP address
const SERVER_PORT = 3000;

const socket = io(`http://${SERVER_IP}:${SERVER_PORT}`);

export default socket;
```

#### Mouse Area Component

```typescript:mobile/src/components/MouseArea.tsx
import React from 'react';
import { View, PanResponder, GestureResponderEvent, PanResponderGestureState } from 'react-native';
import socket from '../socket';

const MouseArea: React.FC = () => {
    const panResponder = React.useRef(
        PanResponder.create({
            onStartShouldSetPanResponder: () => true,
            onPanResponderMove: (evt: GestureResponderEvent, gestureState: PanResponderGestureState) => {
                socket.emit('move', { x: gestureState.moveX, y: gestureState.moveY });
            },
            onPanResponderRelease: () => {
                socket.emit('click', { button: 'left' });
            },
        })
    ).current;

    return <View style={{ flex: 1 }} {...panResponder.panHandlers} />;
};

export default MouseArea;
```

#### App Component

```typescript:mobile/src/App.tsx
import React from 'react';
import { SafeAreaView, StyleSheet } from 'react-native';
import MouseArea from './components/MouseArea';

const App: React.FC = () => {
    return (
        <SafeAreaView style={styles.container}>
            <MouseArea />
        </SafeAreaView>
    );
};

const styles = StyleSheet.create({
    container: {
        flex: 1,
    },
});

export default App;
```

---

## Deployment

### 1. Deploying the Backend Server

#### Packaging the Server

You can package the backend server as an executable using tools like **pkg**.

```bash
npm install -g pkg
pkg .
```

#### Running the Server on Target Machines

Distribute the executable to target machines. Ensure that the required ports are open and accessible over the network.

### 2. Deploying the Mobile Application

#### Android

- Generate a signed APK:
  
  ```bash
  npm run android --variant=release
  ```
  
- Upload to Google Play Store following their [release guide](https://developer.android.com/studio/publish).

#### iOS

- Archive the build in Xcode and upload to the App Store following Apple's [App Store Connect guide](https://developer.apple.com/app-store/connect/).

---

## Testing

1. **Backend Server:**
    - Start the server and ensure it's listening on the specified port.
    - Use tools like Postman to test socket connections.

2. **Mobile Application:**
    - Run the app on a simulator or real device.
    - Connect to the backend server.
    - Test mouse movements and click actions.

3. **End-to-End Testing:**
    - Perform actions on the mobile app and verify corresponding mouse behavior on the PC.
    - Test across different network conditions.

---

## Troubleshooting

- **Connection Issues:**
    - Ensure both devices are on the same network.
    - Verify the server IP and port are correctly configured.
    - Check firewall settings on the PC.

- **Latency or Lag:**
    - Optimize network performance.
    - Consider implementing data throttling or compression.

- **Permission Errors:**
    - On PC, ensure the server has permissions to control the mouse.
    - On mobile, ensure the app has necessary permissions.

- **Cross-Platform Issues:**
    - Test on all intended platforms to identify and fix platform-specific bugs.

---

## Conclusion

By following this guide, developers can quickly build and deploy the **Smartphone as Mouse App**, enabling seamless control of PC mouse functions via smartphones. The cross-platform nature ensures wide accessibility, enhancing user interaction across devices. For further enhancements, consider adding features like multi-button support, gesture customization, and secure authentication mechanisms.

---

# License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

# Contributing

Contributions are welcome! Please open an issue or submit a pull request for any improvements.

# Contact

For any questions or support, please contact [your.email@example.com](mailto:your.email@example.com).

# Acknowledgements

- [React Native](https://reactnative.dev/)
- [Socket.IO](https://socket.io/)
- [RobotJS](https://robotjs.io/)