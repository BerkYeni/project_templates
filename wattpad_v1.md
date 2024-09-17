# Wattpad Clone Development Guide

Welcome to the Wattpad Clone Development Guide! This document provides step-by-step instructions for developers to build and deploy a Wattpad-like application using the following technology stack:

- **Backend:** Express.js
- **Frontend:** React, Tailwind CSS, Shadcn UI
- **Database:** PostgreSQL
- **Authentication:** Google OAuth 2.0
- **Hosting:** Cloudflare VPS

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Setup](#project-setup)
3. [Backend Development](#backend-development)
   - [Initialize Express Server](#initialize-express-server)
   - [Set Up PostgreSQL Database](#set-up-postgresql-database)
   - [Implement Authentication with Google OAuth 2.0](#implement-authentication-with-google-oauth-20)
   - [Define API Endpoints](#define-api-endpoints)
4. [Frontend Development](#frontend-development)
   - [Initialize React Application](#initialize-react-application)
   - [Configure Tailwind CSS and Shadcn UI](#configure-tailwind-css-and-shadcn-ui)
   - [Implement Authentication Flow](#implement-authentication-flow)
   - [Build Core Features](#build-core-features)
5. [Integration](#integration)
6. [Deployment](#deployment)
   - [Prepare Cloudflare VPS](#prepare-cloudflare-vps)
   - [Deploy Backend and Frontend](#deploy-backend-and-frontend)
7. [Folder Structure](#folder-structure)
8. [Code Samples](#code-samples)
9. [Additional Considerations](#additional-considerations)

---

## Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js** (v14 or later)
- **npm** or **yarn**
- **PostgreSQL**
- **Git**
- **Cloudflare Account** with VPS setup
- **Google Cloud Platform Account** for OAuth credentials

---

## Project Setup

1. **Create a Git Repository**

   ```bash
   git init wattpad-clone
   cd wattpad-clone
   ```

2. **Initialize Backend and Frontend Directories**

   ```bash
   mkdir backend frontend
   ```

---

## Backend Development

### Initialize Express Server

1. **Navigate to Backend Directory**

   ```bash
   cd backend
   ```

2. **Initialize npm and Install Dependencies**

   ```bash
   npm init -y
   npm install express dotenv cors pg passport passport-google-oauth20 jsonwebtoken bcrypt
   npm install --save-dev nodemon
   ```

3. **Create Server File**

   - **File Path:** `backend/server.js`

   ```javascript:backend/server.js
   const express = require('express');
   const cors = require('cors');
   const dotenv = require('dotenv');
   const passport = require('passport');
   
   dotenv.config();
   
   const app = express();
   
   // Middleware
   app.use(cors());
   app.use(express.json());
   app.use(passport.initialize());
   
   // Routes
   const authRoutes = require('./routes/auth');
   const storyRoutes = require('./routes/stories');
   
   app.use('/api/auth', authRoutes);
   app.use('/api/stories', storyRoutes);
   
   const PORT = process.env.PORT || 5000;
   
   app.listen(PORT, () => {
       console.log(`Server running on port ${PORT}`);
   });
   ```

### Set Up PostgreSQL Database

1. **Configure Database Connection**

   - **File Path:** `backend/config/db.js`

   ```javascript:backend/config/db.js
   const { Pool } = require('pg');
   const dotenv = require('dotenv');
   
   dotenv.config();
   
   const pool = new Pool({
       connectionString: process.env.DATABASE_URL,
       ssl: {
           rejectUnauthorized: false
       }
   });
   
   module.exports = pool;
   ```

2. **Create Environment Variables**

   - **File Path:** `backend/.env`

   ```
   PORT=5000
   DATABASE_URL=postgresql://username:password@localhost:5432/wattpad_clone
   GOOGLE_CLIENT_ID=your_google_client_id
   GOOGLE_CLIENT_SECRET=your_google_client_secret
   JWT_SECRET=your_jwt_secret
   ```

3. **Set Up Database Schema**

   - **File Path:** `backend/migrations/init.sql`

   ```sql:migrations/init.sql
   CREATE TABLE users (
       id SERIAL PRIMARY KEY,
       google_id VARCHAR(255) UNIQUE NOT NULL,
       name VARCHAR(255) NOT NULL,
       email VARCHAR(255) UNIQUE NOT NULL,
       avatar VARCHAR(255),
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   CREATE TABLE stories (
       id SERIAL PRIMARY KEY,
       user_id INTEGER REFERENCES users(id),
       title VARCHAR(255) NOT NULL,
       description TEXT,
       content TEXT NOT NULL,
       cover_image VARCHAR(255),
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   CREATE TABLE comments (
       id SERIAL PRIMARY KEY,
       story_id INTEGER REFERENCES stories(id),
       user_id INTEGER REFERENCES users(id),
       content TEXT NOT NULL,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

   **Run Migration:**

   ```bash
   psql -U username -d wattpad_clone -f migrations/init.sql
   ```

### Implement Authentication with Google OAuth 2.0

1. **Configure Passport Strategy**

   - **File Path:** `backend/config/passport.js`

   ```javascript:backend/config/passport.js
   const passport = require('passport');
   const GoogleStrategy = require('passport-google-oauth20').Strategy;
   const pool = require('./db');

   passport.use(new GoogleStrategy({
       clientID: process.env.GOOGLE_CLIENT_ID,
       clientSecret: process.env.GOOGLE_CLIENT_SECRET,
       callbackURL: "/api/auth/google/callback"
   },
   async (accessToken, refreshToken, profile, done) => {
       const { id, displayName, emails, photos } = profile;
       try {
           let user = await pool.query('SELECT * FROM users WHERE google_id = $1', [id]);
           if (user.rows.length === 0) {
               // Create new user
               user = await pool.query(
                   'INSERT INTO users (google_id, name, email, avatar) VALUES ($1, $2, $3, $4) RETURNING *',
                   [id, displayName, emails[0].value, photos[0].value]
               );
           }
           return done(null, user.rows[0]);
       } catch (err) {
           return done(err, null);
       }
   }));
   
   module.exports = passport;
   ```

2. **Create Authentication Routes**

   - **File Path:** `backend/routes/auth.js`

   ```javascript:backend/routes/auth.js
   const express = require('express');
   const passport = require('../config/passport');
   const jwt = require('jsonwebtoken');
   
   const router = express.Router();
   
   // Initiate Google OAuth
   router.get('/google',
       passport.authenticate('google', { scope: ['profile', 'email'] })
   );
   
   // Google OAuth callback
   router.get('/google/callback', 
       passport.authenticate('google', { failureRedirect: '/' }),
       (req, res) => {
           // Generate JWT
           const token = jwt.sign({ id: req.user.id }, process.env.JWT_SECRET, { expiresIn: '1h' });
           // Redirect to frontend with token
           res.redirect(`http://localhost:3000/auth?token=${token}`);
       }
   );
   
   module.exports = router;
   ```

### Define API Endpoints

1. **Create Story Routes**

   - **File Path:** `backend/routes/stories.js`

   ```javascript:backend/routes/stories.js
   const express = require('express');
   const jwt = require('jsonwebtoken');
   const pool = require('../config/db');
   
   const router = express.Router();
   
   // Middleware to verify JWT
   const authenticate = (req, res, next) => {
       const token = req.headers['authorization'];
       if (!token) return res.status(401).json({ message: 'No token provided' });
       jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
           if (err) return res.status(500).json({ message: 'Failed to authenticate token' });
           req.userId = decoded.id;
           next();
       });
   };
   
   // Create a new story
   router.post('/', authenticate, async (req, res) => {
       const { title, description, content, cover_image } = req.body;
       try {
           const newStory = await pool.query(
               'INSERT INTO stories (user_id, title, description, content, cover_image) VALUES ($1, $2, $3, $4, $5) RETURNING *',
               [req.userId, title, description, content, cover_image]
           );
           res.json(newStory.rows[0]);
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });
   
   // Get all stories
   router.get('/', async (req, res) => {
       try {
           const stories = await pool.query(
               'SELECT stories.*, users.name as author FROM stories JOIN users ON stories.user_id = users.id ORDER BY created_at DESC'
           );
           res.json(stories.rows);
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });
   
   // Get a single story
   router.get('/:id', async (req, res) => {
       const { id } = req.params;
       try {
           const story = await pool.query(
               'SELECT stories.*, users.name as author FROM stories JOIN users ON stories.user_id = users.id WHERE stories.id = $1',
               [id]
           );
           if (story.rows.length === 0) return res.status(404).json({ message: 'Story not found' });
           res.json(story.rows[0]);
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });
   
   // Update a story
   router.put('/:id', authenticate, async (req, res) => {
       const { id } = req.params;
       const { title, description, content, cover_image } = req.body;
       try {
           // Verify ownership
           const story = await pool.query('SELECT * FROM stories WHERE id = $1', [id]);
           if (story.rows.length === 0) return res.status(404).json({ message: 'Story not found' });
           if (story.rows[0].user_id !== req.userId) {
               return res.status(403).json({ message: 'Unauthorized' });
           }
           const updatedStory = await pool.query(
               'UPDATE stories SET title = $1, description = $2, content = $3, cover_image = $4, updated_at = NOW() WHERE id = $5 RETURNING *',
               [title, description, content, cover_image, id]
           );
           res.json(updatedStory.rows[0]);
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });
   
   // Delete a story
   router.delete('/:id', authenticate, async (req, res) => {
       const { id } = req.params;
       try {
           // Verify ownership
           const story = await pool.query('SELECT * FROM stories WHERE id = $1', [id]);
           if (story.rows.length === 0) return res.status(404).json({ message: 'Story not found' });
           if (story.rows[0].user_id !== req.userId) {
               return res.status(403).json({ message: 'Unauthorized' });
           }
           await pool.query('DELETE FROM stories WHERE id = $1', [id]);
           res.json({ message: 'Story deleted successfully' });
       } catch (err) {
           res.status(500).json({ error: err.message });
       }
   });
   
   module.exports = router;
   ```

---

## Frontend Development

### Initialize React Application

1. **Navigate to Frontend Directory**

   ```bash
   cd ../frontend
   ```

2. **Initialize React App with Vite**

   ```bash
   npm create vite@latest wattpad-clone-frontend -- --template react
   cd wattpad-clone-frontend
   npm install
   ```

### Configure Tailwind CSS and Shadcn UI

1. **Install Tailwind CSS**

   ```bash
   npm install -D tailwindcss postcss autoprefixer
   npx tailwindcss init -p
   ```

2. **Configure `tailwind.config.js`**

   - **File Path:** `frontend/wattpad-clone-frontend/tailwind.config.js`

   ```javascript:frontend/wattpad-clone-frontend/tailwind.config.js
   module.exports = {
     content: [
       "./index.html",
       "./src/**/*.{js,ts,jsx,tsx}",
     ],
     theme: {
       extend: {},
     },
     plugins: [],
   }
   ```

3. **Add Tailwind Directives to CSS**

   - **File Path:** `frontend/wattpad-clone-frontend/src/index.css`

   ```css:frontend/wattpad-clone-frontend/src/index.css
   @tailwind base;
   @tailwind components;
   @tailwind utilities;
   ```

4. **Install Shadcn UI Components**

   ```bash
   npm install @shadcn/ui
   ```

   *Follow Shadcn UI documentation for additional setup if needed.*

### Implement Authentication Flow

1. **Create Authentication Context**

   - **File Path:** `frontend/wattpad-clone-frontend/src/context/AuthContext.jsx`

   ```jsx:frontend/wattpad-clone-frontend/src/context/AuthContext.jsx
   import React, { createContext, useState, useEffect } from 'react';
   
   export const AuthContext = createContext();
   
   export const AuthProvider = ({ children }) => {
       const [user, setUser] = useState(null);
       
       useEffect(() => {
           const urlParams = new URLSearchParams(window.location.search);
           const token = urlParams.get('token');
           if (token) {
               localStorage.setItem('token', token);
               // Decode token or fetch user data
               setUser({ token });
               window.history.replaceState({}, document.title, "/");
           }
       }, []);
       
       return (
           <AuthContext.Provider value={{ user, setUser }}>
               {children}
           </AuthContext.Provider>
       );
   };
   ```

2. **Wrap Application with AuthProvider**

   - **File Path:** `frontend/wattpad-clone-frontend/src/main.jsx`

   ```jsx:frontend/wattpad-clone-frontend/src/main.jsx
   import React from 'react';
   import ReactDOM from 'react-dom/client';
   import App from './App';
   import './index.css';
   import { AuthProvider } from './context/AuthContext';
   
   ReactDOM.createRoot(document.getElementById('root')).render(
     <React.StrictMode>
       <AuthProvider>
         <App />
       </AuthProvider>
     </React.StrictMode>,
   )
   ```

3. **Create Login Button**

   - **File Path:** `frontend/wattpad-clone-frontend/src/components/LoginButton.jsx`

   ```jsx:frontend/wattpad-clone-frontend/src/components/LoginButton.jsx
   import React from 'react';
   
   const LoginButton = () => {
       const handleLogin = () => {
           window.location.href = 'http://localhost:5000/api/auth/google';
       };
       
       return (
           <button
               onClick={handleLogin}
               className="px-4 py-2 bg-blue-500 text-white rounded"
           >
               Login with Google
           </button>
       );
   };
   
   export default LoginButton;
   ```

4. **Add Login Route**

   - **File Path:** `frontend/wattpad-clone-frontend/src/App.jsx`

   ```jsx:frontend/wattpad-clone-frontend/src/App.jsx
   import React, { useContext } from 'react';
   import { AuthContext } from './context/AuthContext';
   import LoginButton from './components/LoginButton';
   
   function App() {
       const { user } = useContext(AuthContext);
       
       return (
           <div className="min-h-screen bg-gray-100">
               {user ? (
                   <div className="container mx-auto p-4">
                       <h1 className="text-2xl">Welcome to Your Wattpad Clone!</h1>
                       {/* Add Routes and Components */}
                   </div>
               ) : (
                   <div className="flex items-center justify-center h-screen">
                       <LoginButton />
                   </div>
               )}
           </div>
       );
   }
   
   export default App;
   ```

### Build Core Features

1. **Create Story Components**

   - **File Path:** `frontend/wattpad-clone-frontend/src/components/StoryList.jsx`

   ```jsx:frontend/wattpad-clone-frontend/src/components/StoryList.jsx
   import React, { useEffect, useState } from 'react';
   
   const StoryList = () => {
       const [stories, setStories] = useState([]);
       
       useEffect(() => {
           fetch('http://localhost:5000/api/stories')
               .then(response => response.json())
               .then(data => setStories(data))
               .catch(err => console.error(err));
       }, []);
       
       return (
           <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
               {stories.map(story => (
                   <div key={story.id} className="bg-white p-4 rounded shadow">
                       <img src={story.cover_image} alt={story.title} className="w-full h-48 object-cover rounded" />
                       <h2 className="text-xl font-bold mt-2">{story.title}</h2>
                       <p className="text-gray-600">by {story.author}</p>
                       <p className="mt-2">{story.description}</p>
                   </div>
               ))}
           </div>
       );
   };
   
   export default StoryList;
   ```

2. **Integrate StoryList into App**

   - **File Path:** `frontend/wattpad-clone-frontend/src/App.jsx`

   ```jsx
   import React, { useContext } from 'react';
   import { AuthContext } from './context/AuthContext';
   import LoginButton from './components/LoginButton';
   import StoryList from './components/StoryList';
   
   function App() {
       const { user } = useContext(AuthContext);
       
       return (
           <div className="min-h-screen bg-gray-100">
               {user ? (
                   <div className="container mx-auto p-4">
                       <h1 className="text-3xl font-bold mb-4">Latest Stories</h1>
                       <StoryList />
                   </div>
               ) : (
                   <div className="flex items-center justify-center h-screen">
                       <LoginButton />
                   </div>
               )}
           </div>
       );
   }
   
   export default App;
   ```

---

## Integration

1. **Set Up CORS**

   Ensure that the backend allows requests from the frontend domain.

   - **File Path:** `backend/server.js`

   ```javascript
   app.use(cors({
       origin: 'http://localhost:3000',
       credentials: true
   }));
   ```

2. **Handle Protected Routes**

   Use JWT tokens to protect routes and fetch user-specific data on the frontend.

---

## Deployment

### Prepare Cloudflare VPS

1. **Set Up Cloudflare Account and VPS**

   - Sign up for a [Cloudflare account](https://www.cloudflare.com/).
   - Purchase a VPS plan suitable for your application.
   - Set up your VPS with a Linux distribution (e.g., Ubuntu).

2. **Configure Domain and DNS**

   - Point your domain to the Cloudflare DNS.
   - Set up necessary DNS records for your backend and frontend.

### Deploy Backend and Frontend

1. **Deploy Backend**

   - **SSH into VPS**

     ```bash
     ssh your_username@your_vps_ip
     ```

   - **Install Dependencies**

     ```bash
     sudo apt update
     sudo apt install nodejs npm
     ```

   - **Clone Repository and Navigate**

     ```bash
     git clone https://github.com/yourusername/wattpad-clone.git
     cd wattpad-clone/backend
     ```

   - **Install Backend Dependencies**

     ```bash
     npm install
     ```

   - **Set Environment Variables**

     - Create a `.env` file with production values.

   - **Run Backend**

     ```bash
     npm run start
     ```

   - **Use a Process Manager (e.g., PM2)**

     ```bash
     npm install -g pm2
     pm2 start server.js
     pm2 startup
     pm2 save
     ```

2. **Deploy Frontend**

   - **Navigate to Frontend Directory**

     ```bash
     cd ../frontend/wattpad-clone-frontend
     ```

   - **Build Frontend**

     ```bash
     npm run build
     ```

   - **Serve Frontend**

     Use a static server or integrate with a web server like Nginx.

   - **Example Using Nginx**

     - **Install Nginx**

       ```bash
       sudo apt install nginx
       ```

     - **Configure Nginx**

       - **File Path:** `/etc/nginx/sites-available/wattpad-clone`

       ```nginx:etc/nginx/sites-available/wattpad-clone
       server {
           listen 80;
           server_name yourdomain.com;
       
           location / {
               root /path/to/wattpad-clone/frontend/wattpad-clone-frontend/dist;
               try_files $uri /index.html;
           }
       
           location /api/ {
               proxy_pass http://localhost:5000/api/;
               proxy_http_version 1.1;
               proxy_set_header Upgrade $http_upgrade;
               proxy_set_header Connection 'upgrade';
               proxy_set_header Host $host;
               proxy_cache_bypass $http_upgrade;
           }
       }
       ```

     - **Enable Configuration and Restart Nginx**

       ```bash
       sudo ln -s /etc/nginx/sites-available/wattpad-clone /etc/nginx/sites-enabled/
       sudo nginx -t
       sudo systemctl restart nginx
       ```

3. **Secure with SSL**

   - Use Cloudflare’s SSL or set up Let’s Encrypt on your VPS.

---

## Folder Structure

```
wattpad-clone/
├── backend/
│   ├── config/
│   │   ├── db.js
│   │   └── passport.js
│   ├── migrations/
│   │   └── init.sql
│   ├── routes/
│   │   ├── auth.js
│   │   └── stories.js
│   ├── .env
│   ├── package.json
│   └── server.js
└── frontend/
    └── wattpad-clone-frontend/
        ├── public/
        ├── src/
        │   ├── components/
        │   │   ├── LoginButton.jsx
        │   │   └── StoryList.jsx
        │   ├── context/
        │   │   └── AuthContext.jsx
        │   ├── App.jsx
        │   ├── main.jsx
        │   └── index.css
        ├── tailwind.config.js
        ├── package.json
        └── vite.config.js
```

---

## Code Samples

### Express Server Setup

- **File Path:** `backend/server.js`

```javascript:backend/server.js
const express = require('express');
const cors = require('cors');
const dotenv = require('dotenv');
const passport = require('passport');

dotenv.config();

const app = express();

// Middleware
app.use(cors({
    origin: 'http://localhost:3000',
    credentials: true
}));
app.use(express.json());
app.use(passport.initialize());

// Routes
const authRoutes = require('./routes/auth');
const storyRoutes = require('./routes/stories');

app.use('/api/auth', authRoutes);
app.use('/api/stories', storyRoutes);

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

### React App Entry Point

- **File Path:** `frontend/wattpad-clone-frontend/src/main.jsx`

```jsx:frontend/wattpad-clone-frontend/src/main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';
import { AuthProvider } from './context/AuthContext';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <AuthProvider>
      <App />
    </AuthProvider>
  </React.StrictMode>,
)
```

### Authentication Context

- **File Path:** `frontend/wattpad-clone-frontend/src/context/AuthContext.jsx`

```jsx:frontend/wattpad-clone-frontend/src/context/AuthContext.jsx
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
    const [user, setUser] = useState(null);
    
    useEffect(() => {
        const urlParams = new URLSearchParams(window.location.search);
        const token = urlParams.get('token');
        if (token) {
            localStorage.setItem('token', token);
            // Optionally decode token or fetch user data
            setUser({ token });
            window.history.replaceState({}, document.title, "/");
        }
    }, []);
    
    return (
        <AuthContext.Provider value={{ user, setUser }}>
            {children}
        </AuthContext.Provider>
    );
};
```

### Story List Component

- **File Path:** `frontend/wattpad-clone-frontend/src/components/StoryList.jsx`

```jsx:frontend/wattpad-clone-frontend/src/components/StoryList.jsx
import React, { useEffect, useState } from 'react';

const StoryList = () => {
    const [stories, setStories] = useState([]);
    
    useEffect(() => {
        fetch('http://localhost:5000/api/stories')
            .then(response => response.json())
            .then(data => setStories(data))
            .catch(err => console.error(err));
    }, []);
    
    return (
        <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
            {stories.map(story => (
                <div key={story.id} className="bg-white p-4 rounded shadow">
                    <img src={story.cover_image} alt={story.title} className="w-full h-48 object-cover rounded" />
                    <h2 className="text-xl font-bold mt-2">{story.title}</h2>
                    <p className="text-gray-600">by {story.author}</p>
                    <p className="mt-2">{story.description}</p>
                </div>
            ))}
        </div>
    );
};

export default StoryList;
```

---

## Additional Considerations

- **Error Handling:** Implement comprehensive error handling on both frontend and backend to enhance user experience.
- **Security:** Ensure secure handling of JWT tokens, use HTTPS, and sanitize inputs to prevent SQL injection.
- **Scalability:** Consider using cloud databases like AWS RDS for PostgreSQL and containerization (e.g., Docker) for easier scaling.
- **Testing:** Implement unit and integration tests to ensure reliability.
- **CI/CD Pipeline:** Set up Continuous Integration and Continuous Deployment pipelines for automated testing and deployment.
- **Responsive Design:** Ensure the frontend is responsive across various devices for better accessibility.

---

Congratulations! You now have a comprehensive guide to building and deploying a Wattpad clone using Express, React, Tailwind CSS, Shadcn UI, PostgreSQL, Google OAuth 2.0, and Cloudflare VPS. Happy coding!