# Pomodoro App Development Guide

This document provides step-by-step instructions to build a Pomodoro App using **Express** for the backend, **SQLite** for the database, **React**, **Tailwind CSS**, and **Shadcn** for the frontend, and **Cloudflare VPS** for hosting. By following this guide, developers can quickly set up, develop, and deploy the application.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Backend Setup](#backend-setup)
    - [Initialize the Express Project](#initialize-the-express-project)
    - [Setup SQLite Database](#setup-sqlite-database)
    - [Define API Routes](#define-api-routes)
3. [Frontend Setup](#frontend-setup)
    - [Initialize the React Project](#initialize-the-react-project)
    - [Setup Tailwind CSS and Shadcn](#setup-tailwind-css-and-shadcn)
    - [Develop Components](#develop-components)
4. [Deployment](#deployment)
    - [Configure Cloudflare VPS](#configure-cloudflare-vps)
    - [Deploy Backend and Frontend](#deploy-backend-and-frontend)
5. [Conclusion](#conclusion)

---

## Prerequisites

Before starting, ensure you have the following installed:

- **Node.js** (v14 or higher)
- **npm** or **yarn**
- **Git**
- **Cloudflare Account** with VPS access
- **Basic knowledge** of JavaScript, React, and Express

---

## Backend Setup

### Initialize the Express Project

1. **Create the Project Directory**

    ```bash
    mkdir pomodoro-app-backend
    cd pomodoro-app-backend
    ```

2. **Initialize npm**

    ```bash
    npm init -y
    ```

3. **Install Dependencies**

    ```bash
    npm install express sqlite3 cors
    npm install --save-dev nodemon
    ```

4. **Setup `package.json` Scripts**

    Update `package.json` to include a start script:

    ```json
    "scripts": {
        "start": "node index.js",
        "dev": "nodemon index.js"
    }
    ```

5. **Create the Entry Point**

    ```bash
    touch index.js
    ```

### Setup SQLite Database

1. **Create the Database Configuration**

    ```bash
    mkdir src
    touch src/db.js
    ```

    ```typescript:src/db.js
    const sqlite3 = require('sqlite3').verbose();
    const path = require('path');

    const dbPath = path.resolve(__dirname, 'pomodoro.db');
    const db = new sqlite3.Database(dbPath, (err) => {
        if (err) {
            console.error('Could not connect to database', err);
        } else {
            console.log('Connected to SQLite database');
        }
    });

    // Initialize tables
    db.serialize(() => {
        db.run(`
            CREATE TABLE IF NOT EXISTS sessions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                task TEXT NOT NULL,
                duration INTEGER NOT NULL,
                completed INTEGER DEFAULT 0,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        `);
    });

    module.exports = db;
    ```

2. **Initialize the Database in `index.js`**

    ```typescript:index.js
    const express = require('express');
    const cors = require('cors');
    const db = require('./src/db');

    const app = express();
    const PORT = process.env.PORT || 5000;

    app.use(cors());
    app.use(express.json());

    // Import routes
    const sessionRoutes = require('./src/routes/sessions');
    app.use('/api/sessions', sessionRoutes);

    app.listen(PORT, () => {
        console.log(`Server is running on port ${PORT}`);
    });
    ```

### Define API Routes

1. **Create Routes Directory and Sessions Route**

    ```bash
    mkdir src/routes
    touch src/routes/sessions.js
    ```

    ```typescript:src/routes/sessions.js
    const express = require('express');
    const router = express.Router();
    const db = require('../db');

    // Get all sessions
    router.get('/', (req, res) => {
        db.all('SELECT * FROM sessions', [], (err, rows) => {
            if (err) {
                res.status(500).json({ error: err.message });
                return;
            }
            res.json({ sessions: rows });
        });
    });

    // Create a new session
    router.post('/', (req, res) => {
        const { task, duration } = req.body;
        const stmt = db.prepare('INSERT INTO sessions (task, duration) VALUES (?, ?)');
        stmt.run(task, duration, function(err) {
            if (err) {
                res.status(500).json({ error: err.message });
                return;
            }
            res.json({ id: this.lastID });
        });
        stmt.finalize();
    });

    // Update a session as completed
    router.put('/:id', (req, res) => {
        const { id } = req.params;
        db.run('UPDATE sessions SET completed = 1 WHERE id = ?', [id], function(err) {
            if (err) {
                res.status(500).json({ error: err.message });
                return;
            }
            res.json({ updated: this.changes });
        });
    });

    // Delete a session
    router.delete('/:id', (req, res) => {
        const { id } = req.params;
        db.run('DELETE FROM sessions WHERE id = ?', [id], function(err) {
            if (err) {
                res.status(500).json({ error: err.message });
                return;
            }
            res.json({ deleted: this.changes });
        });
    });

    module.exports = router;
    ```

---

## Frontend Setup

### Initialize the React Project

1. **Create the Frontend Directory**

    ```bash
    cd ..
    npx create-react-app pomodoro-app-frontend
    cd pomodoro-app-frontend
    ```

2. **Install Dependencies**

    ```bash
    npm install axios
    ```

### Setup Tailwind CSS and Shadcn

1. **Install Tailwind CSS**

    ```bash
    npm install -D tailwindcss postcss autoprefixer
    npx tailwindcss init -p
    ```

2. **Configure `tailwind.config.js`**

    ```typescript:tailwind.config.js
    module.exports = {
        content: [
            "./src/**/*.{js,jsx,ts,tsx}",
        ],
        theme: {
            extend: {},
        },
        plugins: [],
    }
    ```

3. **Add Tailwind Directives to CSS**

    ```css:src/index.css
    @tailwind base;
    @tailwind components;
    @tailwind utilities;
    ```

4. **Install Shadcn UI**

    ```bash
    npm install @shadcn/ui
    ```

    > *Note: Follow specific Shadcn setup instructions if available.*

### Develop Components

1. **Create Component Structure**

    ```bash
    mkdir src/components
    touch src/components/PomodoroTimer.jsx
    touch src/components/SessionList.jsx
    ```

2. **Pomodoro Timer Component**

    ```typescript:src/components/PomodoroTimer.jsx
    import React, { useState, useEffect } from 'react';

    function PomodoroTimer() {
        const [seconds, setSeconds] = useState(1500); // 25 minutes
        const [isActive, setIsActive] = useState(false);

        useEffect(() => {
            let interval = null;
            if (isActive && seconds > 0) {
                interval = setInterval(() => {
                    setSeconds(seconds - 1);
                }, 1000);
            } else if (!isActive && seconds !== 0) {
                clearInterval(interval);
            }

            return () => clearInterval(interval);
        }, [isActive, seconds]);

        const toggle = () => {
            setIsActive(!isActive);
        };

        const reset = () => {
            setSeconds(1500);
            setIsActive(false);
        };

        return (
            <div className="p-4 bg-white rounded shadow">
                <h2 className="text-xl font-bold">Pomodoro Timer</h2>
                <div className="text-5xl my-4">
                    {Math.floor(seconds / 60)}:{('0' + seconds % 60).slice(-2)}
                </div>
                <button onClick={toggle} className="btn btn-primary mr-2">
                    {isActive ? 'Pause' : 'Start'}
                </button>
                <button onClick={reset} className="btn btn-secondary">
                    Reset
                </button>
            </div>
        );
    }

    export default PomodoroTimer;
    ```

3. **Session List Component**

    ```typescript:src/components/SessionList.jsx
    import React, { useEffect, useState } from 'react';
    import axios from 'axios';

    function SessionList() {
        const [sessions, setSessions] = useState([]);

        useEffect(() => {
            fetchSessions();
        }, []);

        const fetchSessions = async () => {
            try {
                const response = await axios.get('http://localhost:5000/api/sessions');
                setSessions(response.data.sessions);
            } catch (error) {
                console.error('Error fetching sessions:', error);
            }
        };

        const markCompleted = async (id) => {
            try {
                await axios.put(`http://localhost:5000/api/sessions/${id}`);
                fetchSessions();
            } catch (error) {
                console.error('Error updating session:', error);
            }
        };

        const deleteSession = async (id) => {
            try {
                await axios.delete(`http://localhost:5000/api/sessions/${id}`);
                fetchSessions();
            } catch (error) {
                console.error('Error deleting session:', error);
            }
        };

        return (
            <div className="p-4 bg-white rounded shadow mt-4">
                <h2 className="text-xl font-bold">Session History</h2>
                <ul>
                    {sessions.map(session => (
                        <li key={session.id} className="flex justify-between items-center my-2">
                            <span>{session.task} - {session.duration} mins</span>
                            <div>
                                {!session.completed && (
                                    <button onClick={() => markCompleted(session.id)} className="btn btn-success mr-2">
                                        Complete
                                    </button>
                                )}
                                <button onClick={() => deleteSession(session.id)} className="btn btn-danger">
                                    Delete
                                </button>
                            </div>
                        </li>
                    ))}
                </ul>
            </div>
        );
    }

    export default SessionList;
    ```

4. **Integrate Components into App**

    ```typescript:src/App.jsx
    import React from 'react';
    import PomodoroTimer from './components/PomodoroTimer';
    import SessionList from './components/SessionList';

    function App() {
        return (
            <div className="container mx-auto p-4">
                <h1 className="text-3xl font-bold text-center mb-6">Pomodoro App</h1>
                <PomodoroTimer />
                <SessionList />
            </div>
        );
    }

    export default App;
    ```

---

## Deployment

### Configure Cloudflare VPS

1. **Provision the VPS**

    - Log in to your Cloudflare account.
    - Navigate to the **VPS** section and provision a new instance.

2. **Access the VPS**

    ```bash
    ssh user@your_vps_ip
    ```

3. **Install Necessary Software**

    ```bash
    # Update packages
    sudo apt update && sudo apt upgrade -y

    # Install Node.js and npm
    curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
    sudo apt install -y nodejs

    # Install Git
    sudo apt install -y git

    # Install PM2 for process management
    sudo npm install -g pm2
    ```

### Deploy Backend and Frontend

1. **Clone the Repository**

    ```bash
    git clone https://github.com/yourusername/pomodoro-app.git
    cd pomodoro-app/backend
    ```

2. **Install Backend Dependencies**

    ```bash
    cd pomodoro-app-backend
    npm install
    ```

3. **Start the Backend with PM2**

    ```bash
    pm2 start index.js --name pomodoro-backend
    pm2 save
    ```

4. **Setup Frontend**

    ```bash
    cd ../frontend
    npm install
    ```

5. **Build the React App**

    ```bash
    npm run build
    ```

6. **Serve the Frontend**

    You can use a static server like `serve` or configure Nginx.

    **Using `serve`:**

    ```bash
    npm install -g serve
    serve -s build -l 3000
    pm2 start serve --name pomodoro-frontend -- -s build -l 3000
    pm2 save
    ```

    **Using Nginx:**

    - **Install Nginx**

        ```bash
        sudo apt install nginx
        ```

    - **Configure Nginx**

        ```bash
        sudo nano /etc/nginx/sites-available/pomodoro
        ```

        ```nginx
        server {
            listen 80;
            server_name your_domain.com;

            location /api/ {
                proxy_pass http://localhost:5000/;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
            }

            location / {
                root /path_to_pomodoro-app-frontend/build;
                try_files $uri /index.html;
            }
        }
        ```

    - **Enable the Configuration**

        ```bash
        sudo ln -s /etc/nginx/sites-available/pomodoro /etc/nginx/sites-enabled/
        sudo nginx -t
        sudo systemctl restart nginx
        ```

7. **Configure Domain and SSL with Cloudflare**

    - Point your domain to the VPS IP in Cloudflare DNS settings.
    - Enable SSL/TLS in Cloudflare for secure connections.

---

## Conclusion

By following this guide, you have set up a full-stack Pomodoro App with a robust backend using Express and SQLite, a responsive frontend with React, Tailwind CSS, and Shadcn, and deployed it on a Cloudflare VPS. Customize and extend the app as needed to enhance functionality and user experience.

---

# Additional Resources

- [Express Documentation](https://expressjs.com/)
- [SQLite Documentation](https://www.sqlite.org/docs.html)
- [React Documentation](https://reactjs.org/docs/getting-started.html)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [Shadcn UI Documentation](https://shadcn/ui)
- [Cloudflare VPS Documentation](https://developers.cloudflare.com/vps/)