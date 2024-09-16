Sure! Below is a comprehensive backend project template that leverages **Node.js** with **TypeScript** for the backend, **PostgreSQL** for the database, and a frontend built with **React**, **TypeScript**, **Tailwind CSS**, and **Shadcn UI**. Additionally, OAuth is integrated for authentication.

---

## Project Structure

```
my-project/
├── backend/
│   ├── src/
│   │   ├── controllers/
│   │   ├── models/
│   │   ├── routes/
│   │   ├── services/
│   │   ├── utils/
│   │   ├── app.ts
│   │   └── server.ts
│   ├── .env
│   ├── tsconfig.json
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   ├── styles/
│   │   ├── App.tsx
│   │   └── index.tsx
│   ├── public/
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   ├── tsconfig.json
│   └── package.json
├── docker-compose.yml
├── README.md
└── .gitignore
```

---

## Backend Setup

### 1. Initialize Backend

```bash
cd backend
npm init -y
npm install express typescript ts-node-dev @types/express pg typeorm dotenv passport passport-oauth2
npx tsc --init
```

### 2. `backend/package.json`

```json
{
  "name": "backend",
  "version": "1.0.0",
  "main": "src/server.ts",
  "scripts": {
    "dev": "ts-node-dev src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.8.0",
    "typeorm": "^0.3.12",
    "dotenv": "^16.0.3",
    "passport": "^0.6.0",
    "passport-oauth2": "^1.6.1"
  },
  "devDependencies": {
    "@types/express": "^4.17.14",
    "typescript": "^4.8.4",
    "ts-node-dev": "^2.0.0"
  }
}
```

### 3. `backend/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES6",
    "module": "commonjs",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

### 4. Environment Variables

Create a `.env` file in the `backend/` directory:

```
PORT=5000
DATABASE_URL=postgres://username:password@localhost:5432/mydatabase
OAUTH_CLIENT_ID=your-client-id
OAUTH_CLIENT_SECRET=your-client-secret
OAUTH_CALLBACK_URL=http://localhost:5000/auth/callback
```

### 5. `backend/src/app.ts`

```typescript:backend/src/app.ts
import express from 'express';
import passport from 'passport';
import session from 'express-session';
import authRoutes from './routes/authRoutes';
import userRoutes from './routes/userRoutes';
import { initializePassport } from './config/passport';

const app = express();

// Middleware
app.use(express.json());
app.use(session({ secret: 'your-secret', resave: false, saveUninitialized: true }));

// Initialize Passport
initializePassport(passport);
app.use(passport.initialize());
app.use(passport.session());

// Routes
app.use('/auth', authRoutes);
app.use('/users', userRoutes);

export default app;
```

### 6. `backend/src/server.ts`

```typescript:backend/src/server.ts
import app from './app';
import { createConnection } from 'typeorm';
import dotenv from 'dotenv';

dotenv.config();

const PORT = process.env.PORT || 5000;

createConnection({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    entities: [__dirname + '/models/*.ts'],
    synchronize: true,
}).then(() => {
    app.listen(PORT, () => {
        console.log(`Server running on port ${PORT}`);
    });
}).catch(error => console.log(error));
```

### 7. Passport Configuration

#### `backend/src/config/passport.ts`

```typescript:backend/src/config/passport.ts
import passport from 'passport';
import { Strategy as OAuth2Strategy } from 'passport-oauth2';
import { Request, Done } from 'express';

export function initializePassport(passport: passport.PassportStatic) {
    passport.use(new OAuth2Strategy({
        authorizationURL: 'https://provider.com/oauth2/authorize',
        tokenURL: 'https://provider.com/oauth2/token',
        clientID: process.env.OAUTH_CLIENT_ID!,
        clientSecret: process.env.OAUTH_CLIENT_SECRET!,
        callbackURL: process.env.OAUTH_CALLBACK_URL!,
    },
    function(accessToken: string, refreshToken: string, profile: any, done: Done) {
        // Here, you'd find or create a user in your database
        return done(null, profile);
    }));

    passport.serializeUser((user, done) => {
        done(null, user);
    });

    passport.deserializeUser((obj: any, done) => {
        done(null, obj);
    });
}
```

### 8. Authentication Routes

#### `backend/src/routes/authRoutes.ts`

```typescript:backend/src/routes/authRoutes.ts
import { Router } from 'express';
import passport from 'passport';

const router = Router();

// Initiate OAuth flow
router.get('/oauth', passport.authenticate('oauth2'));

// OAuth callback
router.get('/callback', 
    passport.authenticate('oauth2', { failureRedirect: '/' }),
    (req, res) => {
        // Successful authentication
        res.redirect('/dashboard');
    }
);

export default router;
```

### 9. User Routes Example

#### `backend/src/routes/userRoutes.ts`

```typescript:backend/src/routes/userRoutes.ts
import { Router } from 'express';
import { getUsers } from '../controllers/userController';

const router = Router();

router.get('/', getUsers);

export default router;
```

### 10. User Controller

#### `backend/src/controllers/userController.ts`

```typescript:backend/src/controllers/userController.ts
import { Request, Response } from 'express';
import { User } from '../models/User';

export const getUsers = async (req: Request, res: Response) => {
    try {
        const users = await User.find();
        res.json(users);
    } catch (err) {
        res.status(500).json({ message: 'Server Error' });
    }
};
```

### 11. User Model

#### `backend/src/models/User.ts`

```typescript:backend/src/models/User.ts
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity()
export class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column({ unique: true })
    email: string;
}
```

---

## Frontend Setup

### 1. Initialize Frontend

```bash
cd frontend
npx create-react-app . --template typescript
npm install tailwindcss postcss autoprefixer @shadcn/ui axios react-router-dom
npx tailwindcss init -p
```

### 2. `frontend/tailwind.config.js`

```javascript:frontend/tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
    "./node_modules/@shadcn/ui/**/*.{js,jsx,ts,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### 3. `frontend/postcss.config.js`

```javascript:frontend/postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

### 4. `frontend/src/index.css`

```css:frontend/src/index.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 5. `frontend/src/App.tsx`

```typescript:frontend/src/App.tsx
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Home from './pages/Home';
import Dashboard from './pages/Dashboard';
import Login from './pages/Login';

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/login" element={<Login />} />
      </Routes>
    </Router>
  );
}

export default App;
```

### 6. Example Page Component

#### `frontend/src/pages/Home.tsx`

```typescript:frontend/src/pages/Home.tsx
import React from 'react';
import { Button } from '@shadcn/ui';

const Home: React.FC = () => {
    return (
        <div className="flex items-center justify-center h-screen bg-gray-100">
            <Button onClick={() => window.location.href='/auth/oauth'}>
                Login with OAuth
            </Button>
        </div>
    );
};

export default Home;
```

### 7. Authentication Service

#### `frontend/src/services/authService.ts`

```typescript:frontend/src/services/authService.ts
import axios from 'axios';

const API_URL = 'http://localhost:5000/auth';

export const loginWithOAuth = () => {
    window.location.href = `${API_URL}/oauth`;
};

export const getUser = async () => {
    const response = await axios.get('http://localhost:5000/users');
    return response.data;
};
```

---

## Docker Configuration

### `docker-compose.yml`

```yaml:docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14
    restart: always
    environment:
      POSTGRES_USER: username
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydatabase
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data

  backend:
    build: ./backend
    restart: always
    env_file:
      - ./backend/.env
    ports:
      - "5000:5000"
    depends_on:
      - db

  frontend:
    build: ./frontend
    restart: always
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  db-data:
```

---

## Getting Started

1. **Clone the Repository**

    ```bash
    git clone https://github.com/your-username/my-project.git
    cd my-project
    ```

2. **Setup Environment Variables**

    - **Backend:** Create a `.env` file in the `backend/` directory as shown above.
    - **Frontend:** Update any necessary environment variables if required.

3. **Start Docker Containers**

    ```bash
    docker-compose up --build
    ```

4. **Access the Application**

    - **Frontend:** [http://localhost:3000](http://localhost:3000)
    - **Backend API:** [http://localhost:5000](http://localhost:5000)

---

## Technologies Used

- **Backend:**
  - Node.js
  - TypeScript
  - Express.js
  - TypeORM
  - PostgreSQL
  - Passport.js (OAuth2)

- **Frontend:**
  - React
  - TypeScript
  - Tailwind CSS
  - Shadcn UI
  - React Router
  - Axios

- **DevOps:**
  - Docker
  - Docker Compose

---

## License

This project is licensed under the MIT License.

---

Feel free to customize this template to fit your specific needs. Happy coding!