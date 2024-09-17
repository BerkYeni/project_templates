# Schulte Table App Building Instructions

Welcome to the **Schulte Table App** development guide! This document provides step-by-step instructions to help developers quickly build and deploy the application using the specified technologies.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Backend Setup (Express & SQLite)](#backend-setup-express--sqlite)
    - [Initialize the Backend Project](#initialize-the-backend-project)
    - [Install Dependencies](#install-dependencies)
    - [Configure SQLite Database](#configure-sqlite-database)
    - [Create API Routes](#create-api-routes)
    - [Start the Backend Server](#start-the-backend-server)
4. [Frontend Setup (React, Tailwind, Shadcn)](#frontend-setup-react-tailwind-shadcn)
    - [Initialize the Frontend Project](#initialize-the-frontend-project)
    - [Install Dependencies](#install-dependencies-1)
    - [Configure Tailwind CSS](#configure-tailwind-css)
    - [Set Up Shadcn UI](#set-up-shadcn-ui)
    - [Create Schulte Table Component](#create-schulte-table-component)
    - [Connect Frontend to Backend](#connect-frontend-to-backend)
5. [Deployment on Cloudflare VPS](#deployment-on-cloudflare-vps)
    - [Set Up Cloudflare VPS](#set-up-cloudflare-vps)
    - [Configure Environment Variables](#configure-environment-variables)
    - [Deploy Backend and Frontend](#deploy-backend-and-frontend)
6. [Additional Considerations](#additional-considerations)
    - [Environment Variables](#environment-variables)
    - [Security Best Practices](#security-best-practices)
    - [Testing](#testing)
7. [Conclusion](#conclusion)

---

## Prerequisites

Before you begin, ensure you have the following installed on your development machine:

- **Node.js** (v14 or later)
- **npm** or **yarn**
- **Git**
- **Cloudflare Account** with VPS access
- **Code Editor** (e.g., VS Code)

---

## Project Structure

```
schulte-table-app/
├── backend/
│   ├── src/
│   │   ├── controllers/
│   │   ├── models/
│   │   ├── routes/
│   │   └── index.js
│   ├── package.json
│   └── .env
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   └── SchulteTable.jsx
│   │   ├── pages/
│   │   ├── App.jsx
│   │   └── index.jsx
│   ├── tailwind.config.js
│   ├── package.json
│   └── .env
├── README.md
└── docker-compose.yml
```

---

## Backend Setup (Express & SQLite)

### Initialize the Backend Project

1. **Navigate to the backend directory:**

    ```bash
    mkdir schulte-table-app
    cd schulte-table-app
    mkdir backend
    cd backend
    ```

2. **Initialize a new Node.js project:**

    ```bash
    npm init -y
    ```

### Install Dependencies

Install the necessary dependencies for Express and SQLite.

```bash
npm install express sqlite3 cors dotenv
npm install --save-dev nodemon
```

### Configure SQLite Database

1. **Create a database file:**

    ```bash
    touch database.sqlite
    ```

2. **Set Up Database Initialization:**

    Create a `models` directory and add a file to initialize the database.

    ```bash
    mkdir src
    mkdir src/models
    touch src/models/db.js
    ```

    ```typescript:backend/src/models/db.js
    const sqlite3 = require('sqlite3').verbose();
    const path = require('path');

    const dbPath = path.resolve(__dirname, '../../database.sqlite');
    const db = new sqlite3.Database(dbPath, (err) => {
        if (err) {
            console.error('Could not connect to database', err);
        } else {
            console.log('Connected to SQLite database');
            db.run(`
                CREATE TABLE IF NOT EXISTS results (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT,
                    score INTEGER,
                    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
                )
            `);
        }
    });

    module.exports = db;
    ```

### Create API Routes

1. **Set Up Routes Directory:**

    ```bash
    mkdir src/routes
    touch src/routes/results.js
    ```

2. **Define Routes:**

    ```typescript:backend/src/routes/results.js
    const express = require('express');
    const router = express.Router();
    const db = require('../models/db');

    // Get all results
    router.get('/', (req, res) => {
        db.all('SELECT * FROM results', [], (err, rows) => {
            if (err) {
                res.status(500).json({ error: err.message });
                return;
            }
            res.json(rows);
        });
    });

    // Add a new result
    router.post('/', (req, res) => {
        const { user_id, score } = req.body;
        const stmt = db.prepare('INSERT INTO results (user_id, score) VALUES (?, ?)');
        stmt.run([user_id, score], function(err) {
            if (err) {
                res.status(500).json({ error: err.message });
                return;
            }
            res.json({ id: this.lastID });
        });
        stmt.finalize();
    });

    module.exports = router;
    ```

3. **Main Server File:**

    ```bash
    touch src/index.js
    ```

    ```typescript:backend/src/index.js
    const express = require('express');
    const cors = require('cors');
    const dotenv = require('dotenv');
    const resultsRouter = require('./routes/results');

    dotenv.config();

    const app = express();
    const PORT = process.env.PORT || 5000;

    app.use(cors());
    app.use(express.json());

    app.use('/api/results', resultsRouter);

    app.get('/', (req, res) => {
        res.send('Schulte Table API');
    });

    app.listen(PORT, () => {
        console.log(`Server is running on port ${PORT}`);
    });
    ```

4. **Configure Nodemon for Development:**

    Update `package.json` scripts:

    ```json
    "scripts": {
        "start": "node src/index.js",
        "dev": "nodemon src/index.js"
    }
    ```

### Start the Backend Server

Start the server in development mode:

```bash
npm run dev
```

The backend should now be running on `http://localhost:5000`.

---

## Frontend Setup (React, Tailwind, Shadcn)

### Initialize the Frontend Project

1. **Navigate to the project root and create frontend directory:**

    ```bash
    cd ..
    npx create-react-app frontend
    cd frontend
    ```

### Install Dependencies

Install Tailwind CSS and Shadcn UI components.

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
npm install @shadcn/ui
```

### Configure Tailwind CSS

1. **Configure `tailwind.config.js`:**

    ```typescript:frontend/tailwind.config.js
    /** @type {import('tailwindcss').Config} */
    module.exports = {
      content: [
        "./src/**/*.{js,jsx,ts,tsx}",
        "./node_modules/@shadcn/ui/**/*.{js,ts,jsx,tsx}",
      ],
      theme: {
        extend: {},
      },
      plugins: [],
    }
    ```

2. **Import Tailwind in `src/index.css`:**

    ```css:frontend/src/index.css
    @tailwind base;
    @tailwind components;
    @tailwind utilities;
    ```

### Set Up Shadcn UI

Follow Shadcn's setup instructions to integrate with React. Ensure components are correctly imported and configured.

### Create Schulte Table Component

1. **Create Components Directory and SchulteTable Component:**

    ```bash
    mkdir src/components
    touch src/components/SchulteTable.jsx
    ```

2. **Implement SchulteTable Component:**

    ```typescript:frontend/src/components/SchulteTable.jsx
    import React, { useState, useEffect } from 'react';

    const SchulteTable = () => {
        const [table, setTable] = useState([]);
        const [currentNumber, setCurrentNumber] = useState(1);
        const [score, setScore] = useState(0);

        useEffect(() => {
            generateTable();
        }, []);

        const generateTable = () => {
            const numbers = Array.from({ length: 25 }, (_, i) => i + 1);
            for (let i = numbers.length - 1; i > 0; i--) {
                const j = Math.floor(Math.random() * (i + 1));
                [numbers[i], numbers[j]] = [numbers[j], numbers[i]];
            }
            setTable(numbers);
        };

        const handleClick = (number) => {
            if (number === currentNumber) {
                setCurrentNumber(currentNumber + 1);
                setScore(score + 1);
            }
        };

        return (
            <div className="flex flex-col items-center">
                <h1 className="text-2xl mb-4">Schulte Table</h1>
                <div className="grid grid-cols-5 gap-4">
                    {table.map((number) => (
                        <button
                            key={number}
                            onClick={() => handleClick(number)}
                            className={`w-16 h-16 flex items-center justify-center border ${
                                number === currentNumber ? 'bg-green-500 text-white' : 'bg-gray-200'
                            }`}
                        >
                            {number}
                        </button>
                    ))}
                </div>
                <p className="mt-4">Score: {score}</p>
                {currentNumber > 25 && <p className="mt-2 text-green-600">Congratulations!</p>}
            </div>
        );
    };

    export default SchulteTable;
    ```

### Connect Frontend to Backend

1. **Create API Service:**

    ```bash
    mkdir src/services
    touch src/services/api.js
    ```

    ```typescript:frontend/src/services/api.js
    const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';

    export const fetchResults = async () => {
        const response = await fetch(`${API_URL}/results`);
        return response.json();
    };

    export const submitResult = async (result) => {
        const response = await fetch(`${API_URL}/results`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(result),
        });
        return response.json();
    };
    ```

2. **Use API in SchulteTable Component:**

    ```typescript:frontend/src/components/SchulteTable.jsx
    // ... previous imports
    import { submitResult } from '../services/api';

    // Inside handleClick
    const handleClick = async (number) => {
        if (number === currentNumber) {
            setCurrentNumber(currentNumber + 1);
            setScore(score + 1);
            if (number === 25) {
                // Submit result to backend
                await submitResult({ user_id: 'user123', score: score + 1 });
            }
        }
    };
    ```

3. **Set API URL in Environment Variables:**

    ```bash
    touch .env
    ```

    ```env:frontend/.env
    REACT_APP_API_URL=http://localhost:5000/api
    ```

### Update App Component

Update `App.jsx` to include the SchulteTable component.

```typescript:frontend/src/App.jsx
import React from 'react';
import SchulteTable from './components/SchulteTable';

function App() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-100">
      <SchulteTable />
    </div>
  );
}

export default App;
```

### Start the Frontend Development Server

```bash
npm start
```

The frontend should now be running on `http://localhost:3000`.

---

## Deployment on Cloudflare VPS

### Set Up Cloudflare VPS

1. **Provision a VPS Instance:**

   - Log in to your Cloudflare account.
   - Navigate to the **Workers** or **R2** service, or use **Cloudflare Pages** for frontend hosting.
   - For a traditional VPS, consider using **Cloudflare's Spectrum** or another suitable service.

2. **Configure DNS Settings:**

   - Point your domain or subdomain to the VPS IP address.
   - Ensure necessary ports (e.g., 80, 443) are open.

### Configure Environment Variables

Set up environment variables for both backend and frontend in the production environment.

```env:backend/.env
PORT=5000
```

```env:frontend/.env
REACT_APP_API_URL=https://your-domain.com/api
```

### Deploy Backend and Frontend

1. **Build the Frontend for Production:**

    ```bash
    cd frontend
    npm run build
    ```

2. **Serve the Frontend with Express (Optional):**

    You can serve the built frontend using Express by configuring static file serving.

    ```typescript:backend/src/index.js
    const path = require('path');

    // After API routes
    app.use(express.static(path.join(__dirname, '../../frontend/build')));

    app.get('*', (req, res) => {
        res.sendFile(path.join(__dirname, '../../frontend/build', 'index.html'));
    });
    ```

3. **Transfer Files to VPS:**

    Use `scp`, `rsync`, or any deployment tool to transfer backend and frontend files to the VPS.

    ```bash
    scp -r ./backend user@your-vps-ip:/path/to/app/backend
    scp -r ./frontend/build user@your-vps-ip:/path/to/app/frontend/build
    ```

4. **Install Dependencies on VPS:**

    ```bash
    cd /path/to/app/backend
    npm install
    ```

5. **Start the Backend Server:**

    Use a process manager like **PM2** to run the backend.

    ```bash
    npm install -g pm2
    pm2 start src/index.js --name schulte-backend
    pm2 save
    ```

6. **Configure Nginx as a Reverse Proxy (Optional):**

    Install and configure Nginx to route traffic to the backend and serve static frontend files.

    ```nginx
    server {
        listen 80;
        server_name your-domain.com;

        location /api/ {
            proxy_pass http://localhost:5000/api/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        location / {
            root /path/to/app/frontend/build;
            try_files $uri /index.html;
        }
    }
    ```

    Restart Nginx:

    ```bash
    sudo systemctl restart nginx
    ```

7. **Secure the Application with SSL:**

    Use **Let's Encrypt** or Cloudflare's SSL features to secure your application.

    ```bash
    sudo apt-get install certbot python3-certbot-nginx
    sudo certbot --nginx -d your-domain.com
    ```

### Verify Deployment

- **Access the App:** Navigate to `https://your-domain.com` to see the Schulte Table App in action.
- **API Endpoints:** Ensure API endpoints like `https://your-domain.com/api/results` are accessible.

---

## Additional Considerations

### Environment Variables

- **Backend:** Use the `.env` file to store sensitive information like database URLs, API keys, etc.
- **Frontend:** Prefix environment variables with `REACT_APP_` to expose them to the frontend.

### Security Best Practices

- **CORS:** Ensure CORS is properly configured to allow only trusted origins.
- **Input Validation:** Validate all incoming data on the backend to prevent SQL injection and other attacks.
- **HTTPS:** Always serve your application over HTTPS to encrypt data in transit.
- **Secrets Management:** Do not commit `.env` files to version control. Use environment variables or secret management tools.

### Testing

Implement testing to ensure the application functions as expected.

- **Backend Testing:** Use tools like **Jest** and **Supertest** to test API endpoints.
- **Frontend Testing:** Use **React Testing Library** and **Jest** for component testing.

---

## Conclusion

You have now set up a complete **Schulte Table App** with an Express backend, SQLite database, React frontend styled with Tailwind and Shadcn UI, and deployed it on a Cloudflare VPS. This guide provides the foundational steps required to get your application up and running. For further enhancements, consider adding user authentication, real-time features, and more comprehensive analytics.

Happy Coding!