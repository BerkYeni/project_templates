# Hobby Sport Score Tracker App - Building Instructions

## Table of Contents
1. [Introduction](#introduction)
2. [Features](#features)
3. [Tech Stack](#tech-stack)
4. [Architecture Overview](#architecture-overview)
5. [Prerequisites](#prerequisites)
6. [Project Setup](#project-setup)
7. [Project Structure](#project-structure)
8. [Backend Development](#backend-development)
    - [API Endpoints](#api-endpoints)
    - [Database Schema](#database-schema)
9. [Frontend Development](#frontend-development)
    - [State Management](#state-management)
    - [UI Components](#ui-components)
10. [Cross-Platform Considerations](#cross-platform-considerations)
11. [Deployment](#deployment)
12. [Testing](#testing)
13. [Future Enhancements](#future-enhancements)
14. [Conclusion](#conclusion)

---

## Introduction

The **Hobby Sport Score Tracker App** allows users to track scores across various hobby sports. Designed to be highly generalized and expandable, the app supports multiple sports and can easily incorporate new ones in the future. It is built as a cross-platform application, ensuring compatibility with major operating systems.

## Features

- **Multi-Sport Support**: Track scores for various hobby sports.
- **User Authentication**: Secure user login and registration.
- **Score Management**: Add, edit, delete, and view scores.
- **Statistics & Insights**: Visualize performance over time.
- **Responsive Design**: Seamless experience across devices.
- **Expandable Architecture**: Easily add support for new sports.

## Tech Stack

- **Frontend**: React Native
- **Backend**: Node.js with Express
- **Database**: MongoDB
- **Authentication**: JWT (JSON Web Tokens)
- **State Management**: Redux
- **Styling**: Styled Components or Tailwind CSS
- **Testing**: Jest, React Native Testing Library
- **Deployment**: AWS or Firebase

## Architecture Overview

The application follows a **Modular MVC** (Model-View-Controller) architecture to ensure separation of concerns, scalability, and ease of maintenance.

- **Models**: Define the data structures (e.g., Users, Sports, Scores).
- **Views**: UI components built with React Native.
- **Controllers**: Handle business logic and API interactions.

![Architecture Diagram](path/to/architecture-diagram.png)

## Prerequisites

- **Node.js** (v14 or above)
- **npm** or **Yarn**
- **MongoDB** instance
- **React Native CLI**
- **Git** for version control

## Project Setup

### 1. Clone the Repository
```bash
git clone https://github.com/yourusername/hobby-sport-score-tracker.git
cd hobby-sport-score-tracker
```

### 2. Install Dependencies

#### Backend
```bash
cd backend
npm install
```

#### Frontend
```bash
cd ../frontend
npm install
```

### 3. Environment Variables

Create a `.env` file in both `backend` and `frontend` directories with the necessary configurations.

#### Backend `.env` Example
```env
PORT=5000
MONGO_URI=your_mongodb_uri
JWT_SECRET=your_jwt_secret
```

#### Frontend `.env` Example
```env
API_URL=http://localhost:5000/api
```

## Project Structure

```plaintext
hobby-sport-score-tracker/
├── backend/
│   ├── controllers/
│   ├── models/
│   ├── routes/
│   ├── middleware/
│   ├── config/
│   ├── app.js
│   └── server.js
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── screens/
│   │   ├── redux/
│   │   ├── navigation/
│   │   ├── services/
│   │   └── App.js
│   └── package.json
├── README.md
└── .gitignore
```

## Backend Development

### Setting Up the Server

```typescript:backend/app.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const userRoutes = require('./routes/userRoutes');
const sportRoutes = require('./routes/sportRoutes');

function createApp() {
    const app = express();
    app.use(cors());
    app.use(express.json());

    // Routes
    app.use('/api/users', userRoutes);
    app.use('/api/sports', sportRoutes);

    return app;
}

module.exports = createApp;
```

### Starting the Server

```typescript:backend/server.js
const createApp = require('./app');
const mongoose = require('mongoose');
require('dotenv').config();

const PORT = process.env.PORT || 5000;

mongoose.connect(process.env.MONGO_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
})
.then(() => {
    console.log('MongoDB connected');
    const app = createApp();
    app.listen(PORT, () => {
        console.log(`Server running on port ${PORT}`);
    });
})
.catch(err => {
    console.error('Database connection error:', err);
});
```

### API Endpoints

#### User Routes

```typescript:backend/routes/userRoutes.js
const express = require('express');
const { registerUser, loginUser, getUserProfile } = require('../controllers/userController');
const { protect } = require('../middleware/authMiddleware');

const router = express.Router();

router.post('/register', registerUser);
router.post('/login', loginUser);
router.get('/profile', protect, getUserProfile);

module.exports = router;
```

#### Sport Routes

```typescript:backend/routes/sportRoutes.js
const express = require('express');
const { getSports, addSport } = require('../controllers/sportController');
const { protect } = require('../middleware/authMiddleware');

const router = express.Router();

router.get('/', protect, getSports);
router.post('/', protect, addSport);

module.exports = router;
```

### Database Schema

#### User Model

```typescript:backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = mongoose.Schema({
    username: { type: String, required: true, unique: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true },
}, {
    timestamps: true,
});

// Password hashing
userSchema.pre('save', async function(next) {
    if (!this.isModified('password')) return next();
    const salt = await bcrypt.genSalt(10);
    this.password = await bcrypt.hash(this.password, salt);
    next();
});

// Password verification
userSchema.methods.matchPassword = async function(enteredPassword) {
    return await bcrypt.compare(enteredPassword, this.password);
}

function User() {
    return mongoose.model('User', userSchema);
}

module.exports = User();
```

#### Sport Model

```typescript:backend/models/Sport.js
const mongoose = require('mongoose');

const sportSchema = mongoose.Schema({
    name: { type: String, required: true, unique: true },
    description: { type: String },
    createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
}, {
    timestamps: true,
});

function Sport() {
    return mongoose.model('Sport', sportSchema);
}

module.exports = Sport();
```

### Controllers

#### User Controller

```typescript:backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Generate JWT
const generateToken = (id) => {
    return jwt.sign({ id }, process.env.JWT_SECRET, {
        expiresIn: '30d',
    });
}

// Register User
const registerUser = async (req, res) => {
    const { username, email, password } = req.body;

    try {
        const userExists = await User.findOne({ email });

        if (userExists) {
            return res.status(400).json({ message: 'User already exists' });
        }

        const user = await User.create({
            username,
            email,
            password,
        });

        if (user) {
            res.status(201).json({
                _id: user._id,
                username: user.username,
                email: user.email,
                token: generateToken(user._id),
            });
        } else {
            res.status(400).json({ message: 'Invalid user data' });
        }
    } catch (error) {
        res.status(500).json({ message: error.message });
    }
}

// Login User
const loginUser = async (req, res) => {
    const { email, password } = req.body;

    try {
        const user = await User.findOne({ email });

        if (user && (await user.matchPassword(password))) {
            res.json({
                _id: user._id,
                username: user.username,
                email: user.email,
                token: generateToken(user._id),
            });
        } else {
            res.status(401).json({ message: 'Invalid email or password' });
        }
    } catch (error) {
        res.status(500).json({ message: error.message });
    }
}

// Get User Profile
const getUserProfile = async (req, res) => {
    try {
        const user = await User.findById(req.user.id).select('-password');
        if (user) {
            res.json(user);
        } else {
            res.status(404).json({ message: 'User not found' });
        }
    } catch(error) {
        res.status(500).json({ message: error.message });
    }
}

module.exports = {
    registerUser,
    loginUser,
    getUserProfile,
}
```

#### Sport Controller

```typescript:backend/controllers/sportController.js
const Sport = require('../models/Sport');

// Get All Sports
const getSports = async (req, res) => {
    try {
        const sports = await Sport.find({ createdBy: req.user.id });
        res.json(sports);
    } catch (error) {
        res.status(500).json({ message: error.message });
    }
}

// Add New Sport
const addSport = async (req, res) => {
    const { name, description } = req.body;

    try {
        const sportExists = await Sport.findOne({ name });

        if (sportExists) {
            return res.status(400).json({ message: 'Sport already exists' });
        }

        const sport = await Sport.create({
            name,
            description,
            createdBy: req.user.id,
        });

        res.status(201).json(sport);
    } catch (error) {
        res.status(500).json({ message: error.message });
    }
}

module.exports = {
    getSports,
    addSport,
}
```

### Middleware

#### Authentication Middleware

```typescript:backend/middleware/authMiddleware.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
    let token;

    if (
        req.headers.authorization && 
        req.headers.authorization.startsWith('Bearer')
    ) {
        try {
            token = req.headers.authorization.split(' ')[1];
            const decoded = jwt.verify(token, process.env.JWT_SECRET);

            req.user = await User.findById(decoded.id).select('-password');
            next();
        } catch (error) {
            res.status(401).json({ message: 'Not authorized, token failed' });
        }
    }

    if (!token) {
        res.status(401).json({ message: 'Not authorized, no token' });
    }
}

module.exports = { protect };
```

## Frontend Development

### Setting Up React Native

Initialize a new React Native project using Expo or React Native CLI.

```bash
npx react-native init HobbySportScoreTracker
cd HobbySportScoreTracker
```

### State Management with Redux

```typescript:frontend/src/redux/store.js
import { createStore, combineReducers, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import { userReducer } from './reducers/userReducer';
import { sportReducer } from './reducers/sportReducer';

const rootReducer = combineReducers({
    user: userReducer,
    sports: sportReducer,
});

const store = createStore(rootReducer, applyMiddleware(thunk));

export default store;
```

### UI Components

#### Header Component

```typescript:frontend/src/components/Header.js
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

function Header({ title }) {
    return (
        <View style={styles.header}>
            <Text style={styles.title}>{title}</Text>
        </View>
    );
}

const styles = StyleSheet.create({
    header: {
        height: 60,
        padding: 15,
        backgroundColor: '#f7287b',
    },
    title: {
        color: 'white',
        fontSize: 23,
        textAlign: 'center',
    },
});

export default Header;
```

#### Sport List Screen

```typescript:frontend/src/screens/SportListScreen.js
import React, { useEffect } from 'react';
import { View, FlatList, Text, StyleSheet } from 'react-native';
import { useDispatch, useSelector } from 'react-redux';
import { fetchSports } from '../redux/actions/sportActions';
import SportItem from '../components/SportItem';

function SportListScreen() {
    const dispatch = useDispatch();
    const sports = useSelector(state => state.sports.items);
    const loading = useSelector(state => state.sports.loading);
    const error = useSelector(state => state.sports.error);

    useEffect(() => {
        dispatch(fetchSports());
    }, [dispatch]);

    if (loading) return <Text>Loading...</Text>;
    if (error) return <Text>Error: {error}</Text>;

    return (
        <View style={styles.container}>
            <FlatList
                data={sports}
                keyExtractor={item => item._id}
                renderItem={({ item }) => <SportItem sport={item} />}
            />
        </View>
    );
}

const styles = StyleSheet.create({
    container: {
        flex: 1,
        padding: 20,
    },
});

export default SportListScreen;
```

### Services

#### API Service

```typescript:frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.API_URL || 'http://localhost:5000/api';

const api = axios.create({
    baseURL: API_URL,
});

// Add authorization header
api.interceptors.request.use(config => {
    const token = localStorage.getItem('token');
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
}, error => Promise.reject(error));

export default api;
```

## Cross-Platform Considerations

- **Responsive Design**: Utilize Flexbox and percentage-based dimensions for adaptable layouts.
- **Platform-Specific Code**: Use `Platform` module from React Native to handle iOS and Android differences.
- **Testing on Multiple Devices**: Ensure testing on both iOS and Android simulators/emulators.
- **Packaging**: Use tools like Expo for easier cross-platform builds or configure native builds accordingly.

## Deployment

### Backend Deployment

1. **Choose a Hosting Provider**: AWS (EC2), Heroku, DigitalOcean, etc.
2. **Set Up Environment Variables**: Ensure `.env` variables are set on the server.
3. **Database Connection**: Configure MongoDB connection (use MongoDB Atlas for managed DB).
4. **Start the Server**: Use process managers like PM2.

```bash
cd backend
npm install -g pm2
pm2 start server.js --name hobby-sport-backend
```

### Frontend Deployment

For React Native:

1. **iOS**: Use Xcode to build and submit to the App Store.
2. **Android**: Use Android Studio to build and submit to Google Play.
3. **CI/CD**: Implement pipelines using tools like Bitrise or GitHub Actions for automated builds.

### Environment Configuration

Ensure that the frontend API URL points to the deployed backend server.

## Testing

### Backend Testing

- **Unit Tests**: Test individual functions and controllers using Jest.
- **Integration Tests**: Test API endpoints with tools like Supertest.

```typescript:backend/tests/user.test.js
const request = require('supertest');
const app = require('../app');
const mongoose = require('mongoose');

describe('User API', () => {
    beforeAll(async () => {
        // Connect to test DB
    });

    afterAll(async () => {
        // Disconnect DB
    });

    it('should register a new user', async () => {
        const res = await request(app)
            .post('/api/users/register')
            .send({
                username: 'testuser',
                email: 'test@example.com',
                password: 'password123',
            });
        expect(res.statusCode).toEqual(201);
        expect(res.body).toHaveProperty('token');
    });
});
```

### Frontend Testing

- **Component Tests**: Use React Native Testing Library to test UI components.
- **Redux Tests**: Test reducers and actions.
- **End-to-End Tests**: Use Detox for automated UI testing.

```typescript:frontend/src/__tests__/Header.test.js
import React from 'react';
import { render } from '@testing-library/react-native';
import Header from '../components/Header';

test('renders header with title', () => {
    const { getByText } = render(<Header title="Hobby Sport Tracker" />);
    expect(getByText('Hobby Sport Tracker')).toBeTruthy();
});
```

## Future Enhancements

- **Real-Time Updates**: Implement WebSockets for live score updates.
- **Social Features**: Allow users to share scores and achievements.
- **Notifications**: Send reminders and updates via push notifications.
- **Advanced Analytics**: Provide deeper insights into performance trends.
- **Multi-language Support**: Internationalize the app for different languages.
- **Offline Support**: Enable offline data access and synchronization.

## Conclusion

This document provides a comprehensive guide to building a **Hobby Sport Score Tracker App**. By following the outlined steps and utilizing the provided code snippets, developers can efficiently implement a robust, scalable, and cross-platform application. The architecture is designed to accommodate future enhancements, ensuring the app remains versatile and adaptable to evolving user needs.

# Quick Start

1. **Clone the Repository**:
    ```bash
    git clone https://github.com/yourusername/hobby-sport-score-tracker.git
    cd hobby-sport-score-tracker
    ```

2. **Set Up Backend**:
    ```bash
    cd backend
    npm install
    cp .env.example .env
    # Fill in the .env with your configurations
    npm run dev
    ```

3. **Set Up Frontend**:
    ```bash
    cd frontend
    npm install
    cp .env.example .env
    # Fill in the .env with your API URL
    npm start
    ```

4. **Run on Devices**:
    - **iOS**: Use Xcode or Expo.
    - **Android**: Use Android Studio or Expo.

---

For detailed instructions, refer to each section above.