# Full-Stack Project Template

This template provides a comprehensive setup for building a full-stack application using **Node.js** with **Express** for the backend, **MongoDB** for the database, and **Vue.js** with **Tailwind CSS** and **Shadcn** for the frontend. **OAuth** is integrated for authentication.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Backend Setup](#backend-setup)
   - [1. Initialize Backend Project](#1-initialize-backend-project)
   - [2. Install Dependencies](#2-install-dependencies)
   - [3. Configure Environment Variables](#3-configure-environment-variables)
   - [4. Setup Server](#4-setup-server)
   - [5. Configure Passport for OAuth](#5-configure-passport-for-oauth)
   - [6. Define User Model](#6-define-user-model)
   - [7. Create Authentication Routes](#7-create-authentication-routes)
   - [8. Create User Routes](#8-create-user-routes)
   - [9. Install Optional Utilities](#9-install-optional-utilities)
3. [Frontend Setup](#frontend-setup)
   - [1. Initialize Frontend Project](#1-initialize-frontend-project)
   - [2. Install Dependencies](#2-install-dependencies-frontend)
   - [3. Configure Tailwind CSS](#3-configure-tailwind-css)
   - [4. Include Tailwind in CSS](#4-include-tailwind-in-css)
   - [5. Configure Shadcn Components](#5-configure-shadcn-components)
   - [6. Setup OAuth Integration](#6-setup-oauth-integration)
   - [7. Example Components](#7-example-components)
   - [8. Configure CORS](#8-configure-cors)
   - [9. Setup Vue Router and Vuex Store](#9-setup-vue-router-and-vuex-store)
4. [Running the Project](#running-the-project)
5. [Deployment](#deployment)
6. [Conclusion](#conclusion)

---

## Project Structure

```plaintext
my-project/
├── backend/
│   ├── config/
│   │   └── passport.js
│   ├── controllers/
│   ├── models/
│   │   └── User.js
│   ├── routes/
│   │   ├── auth.js
│   │   └── user.js
│   ├── app.js
│   └── package.json
├── frontend/
│   ├── public/
│   ├── src/
│   │   ├── assets/
│   │   ├── components/
│   │   │   ├── Login.vue
│   │   │   └── UserProfile.vue
│   │   ├── router/
│   │   │   └── index.js
│   │   ├── store/
│   │   │   └── index.js
│   │   ├── App.vue
│   │   ├── main.js
│   │   └── main.css
│   ├── tailwind.config.js
│   ├── shadcn.config.js
│   └── package.json
├── .env
├── README.md
└── package.json
```

---

## Backend Setup

### 1. Initialize Backend Project

Navigate to the `backend` directory and initialize a Node.js project.

```bash
cd backend
npm init -y
```

### 2. Install Dependencies

```bash
npm install express mongoose passport passport-oauth2 dotenv cors express-session mongoose-findorcreate
```

### 3. Configure Environment Variables

Create a `.env` file at the root of your project:

```dotenv
# .env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
OAUTH_CLIENT_ID=your_oauth_client_id
OAUTH_CLIENT_SECRET=your_oauth_client_secret
OAUTH_CALLBACK_URL=http://localhost:5000/auth/oauth/callback
SESSION_SECRET=your_session_secret
CLIENT_URL=http://localhost:3000
```

### 4. Setup Server

Create `backend/app.js`:

```javascript:backend/app.js
const express = require('express');
const mongoose = require('mongoose');
const passport = require('passport');
const session = require('express-session');
const cors = require('cors');
const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/user');
require('dotenv').config();
require('./config/passport');

const app = express();

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.log(err));

// Middleware
app.use(express.json());
app.use(cors({
    origin: process.env.CLIENT_URL,
    credentials: true,
}));
app.use(session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
}));
app.use(passport.initialize());
app.use(passport.session());

// Routes
app.use('/auth', authRoutes);
app.use('/api/user', userRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### 5. Configure Passport for OAuth

Create `backend/config/passport.js`:

```javascript:backend/config/passport.js
const passport = require('passport');
const OAuth2Strategy = require('passport-oauth2').Strategy;
const User = require('../models/User');

passport.serializeUser(function(user, done) {
    done(null, user.id);
});

passport.deserializeUser(function(id, done) {
    User.findById(id, function(err, user) {
        done(err, user);
    });
});

passport.use('oauth', new OAuth2Strategy({
    authorizationURL: 'https://provider.com/oauth2/authorize', // Replace with actual URL
    tokenURL: 'https://provider.com/oauth2/token', // Replace with actual URL
    clientID: process.env.OAUTH_CLIENT_ID,
    clientSecret: process.env.OAUTH_CLIENT_SECRET,
    callbackURL: process.env.OAUTH_CALLBACK_URL,
},
function(accessToken, refreshToken, profile, done) {
    // Retrieve user profile from provider
    // You might need to make an API call to get user info
    // For simplicity, assuming profile contains id, name, and email
    User.findOrCreate({ oauthId: profile.id }, {
        name: profile.name,
        email: profile.email,
    }, function (err, user) {
        return done(err, user);
    });
}
));
```

> **Note:** Replace `'https://provider.com/oauth2/authorize'` and `'https://provider.com/oauth2/token'` with the actual OAuth provider URLs (e.g., GitHub, Google).

### 6. Define User Model

Create `backend/models/User.js`:

```javascript:backend/models/User.js
const mongoose = require('mongoose');
const findOrCreate = require('mongoose-findorcreate');

const UserSchema = new mongoose.Schema({
    oauthId: {
        type: String,
        required: true,
        unique: true,
    },
    name: {
        type: String,
    },
    email: {
        type: String,
    },
    // Add additional fields as needed
});

UserSchema.plugin(findOrCreate);

module.exports = mongoose.model('User', UserSchema);
```

### 7. Create Authentication Routes

Create `backend/routes/auth.js`:

```javascript:backend/routes/auth.js
const express = require('express');
const passport = require('passport');
const router = express.Router();

// Initiate OAuth Login
router.get('/oauth', passport.authenticate('oauth'));

// OAuth Callback
router.get('/oauth/callback', 
    passport.authenticate('oauth', { failureRedirect: 'http://localhost:3000/login' }),
    function(req, res) {
        // Successful authentication, redirect to client
        res.redirect('http://localhost:3000');
    }
);

// Logout
router.get('/logout', function(req, res){
    req.logout(function(err) {
        if (err) { return next(err); }
        res.redirect('http://localhost:3000');
    });
});

module.exports = router;
```

### 8. Create User Routes

Create `backend/routes/user.js`:

```javascript:backend/routes/user.js
const express = require('express');
const router = express.Router();

// Middleware to check authentication
function isAuthenticated(req, res, next) {
    if (req.isAuthenticated()) return next();
    res.status(401).json({ message: 'Unauthorized' });
}

// Get current user
router.get('/me', isAuthenticated, (req, res) => {
    res.json({
        id: req.user._id,
        name: req.user.name,
        email: req.user.email,
    });
});

module.exports = router;
```

### 9. Install Optional Utilities

To use `findOrCreate`, ensure you've installed the `mongoose-findorcreate` plugin as shown in step 2.

---

## Frontend Setup

### 1. Initialize Frontend Project

Navigate to the `frontend` directory and initialize a Vue project using Vue CLI or Vite. Here, we'll use Vite for faster builds.

```bash
cd frontend
npm create vite@latest
# Follow the prompts:
# Project name: frontend
# Select a framework: Vue
# Select a variant: Vue
```

### 2. Install Dependencies Frontend

Install **Tailwind CSS**, **Shadcn**, and **Axios** for HTTP requests.

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
npm install @headlessui/vue axios
# Note: Shadcn components are primarily for React. For Vue, you can use Headless UI which integrates well with Tailwind.
```

### 3. Configure Tailwind CSS

Modify `frontend/tailwind.config.js`:

```javascript:frontend/tailwind.config.js
module.exports = {
  content: [
    "./index.html",
    "./src/**/*.{vue,js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### 4. Include Tailwind in CSS

Edit `frontend/src/main.css`:

```css:frontend/src/main.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 5. Configure Shadcn Components

Since Shadcn is tailored for React, we'll use **Headless UI** for Vue which offers similar unstyled components.

```bash
npm install @headlessui/vue
```

### 6. Setup OAuth Integration

Implement authentication flow using OAuth endpoints from the backend.

### 7. Example Components

#### a. Login Component

Create `frontend/src/components/Login.vue`:

```vue:frontend/src/components/Login.vue
<template>
  <button @click="login" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">
    Login with OAuth
  </button>
</template>

<script>
export default {
  name: 'Login',
  methods: {
    login() {
      window.location.href = 'http://localhost:5000/auth/oauth';
    }
  }
}
</script>

<style scoped>
/* Add any component-specific styles here */
</style>
```

#### b. User Profile Component

Create `frontend/src/components/UserProfile.vue`:

```vue:frontend/src/components/UserProfile.vue
<template>
  <div v-if="user" class="p-4">
    <h1 class="text-2xl font-bold">{{ user.name }}</h1>
    <p class="text-gray-600">{{ user.email }}</p>
    <button @click="logout" class="mt-4 bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded">
      Logout
    </button>
  </div>
  <div v-else class="p-4">
    <Login />
  </div>
</template>

<script>
import axios from 'axios'
import Login from './Login.vue'

export default {
  name: 'UserProfile',
  components: { Login },
  data() {
    return {
      user: null,
    }
  },
  mounted() {
    this.fetchUser();
  },
  methods: {
    async fetchUser() {
      try {
        const res = await axios.get('http://localhost:5000/api/user/me', { withCredentials: true });
        this.user = res.data;
      } catch (err) {
        this.user = null;
      }
    },
    logout() {
      window.location.href = 'http://localhost:5000/auth/logout';
    }
  }
}
</script>

<style scoped>
/* Add any component-specific styles here */
</style>
```

#### c. App.vue

Update `frontend/src/App.vue` to include the `UserProfile` component.

```vue:frontend/src/App.vue
<template>
  <div class="min-h-screen bg-gray-100 flex items-center justify-center">
    <UserProfile />
  </div>
</template>

<script>
import UserProfile from './components/UserProfile.vue'

export default {
  name: 'App',
  components: {
    UserProfile
  }
}
</script>

<style>
/* Global styles if any */
</style>
```

### 8. Configure CORS

Ensure the backend `app.js` has CORS enabled as shown in the backend setup. This allows the frontend to communicate with the backend.

### 9. Setup Vue Router and Vuex Store

For this template, basic state management and routing are optional. You can set them up as needed.

---

## Running the Project

### 1. Start MongoDB

Ensure MongoDB is running locally or use a cloud service like [MongoDB Atlas](https://www.mongodb.com/cloud/atlas).

### 2. Start Backend Server

In the `backend` directory:

```bash
node app.js
```

Or, for development with auto-reloading:

```bash
npm install -D nodemon
npx nodemon app.js
```

### 3. Start Frontend Server

In the `frontend` directory:

```bash
npm install
npm run dev
```

Navigate to `http://localhost:3000` to view the frontend.

---

## Deployment

### Backend Deployment

- **Platform:** [Heroku](https://www.heroku.com/), [DigitalOcean](https://www.digitalocean.com/), [AWS](https://aws.amazon.com/), etc.
- **Steps:**
  1. Set environment variables on the hosting platform.
  2. Ensure MongoDB is accessible (use MongoDB Atlas for managed databases).
  3. Deploy using Git or CI/CD pipelines.

### Frontend Deployment

- **Platform:** [Vercel](https://vercel.com/), [Netlify](https://www.netlify.com/), [Firebase Hosting](https://firebase.google.com/products/hosting), etc.
- **Steps:**
  1. Build the project: `npm run build`
  2. Deploy the `dist` folder to the hosting platform.
  3. Ensure the frontend is configured to communicate with the deployed backend.

### Environment Variables

Ensure all environment variables are correctly set in both backend and frontend environments, especially OAuth credentials and API endpoints.

---

## Conclusion

This template offers a solid foundation for building scalable and maintainable full-stack applications with modern technologies. Customize and expand upon this setup to fit the specific needs of your project.

**Next Steps:**

- Integrate specific OAuth providers (e.g., Google, GitHub) by updating the Passport strategy.
- Enhance security measures (e.g., HTTPS, secure cookies).
- Implement additional features like role-based access control, data validation, and error handling.
- Optimize frontend components using Shadcn or alternative Vue-compatible component libraries.

Happy Coding!