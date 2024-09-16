# Full-Stack Project Template

This project template provides a robust starting point for developers to quickly build and deploy applications using **FastAPI** for the backend, **PostgreSQL** for the database, and a **React** frontend powered by **TypeScript**, **Tailwind CSS**, and **Shadcn** components. Authentication is managed via **OAuth**.

## Table of Contents

- [Project Structure](#project-structure)
- [Backend Setup](#backend-setup)
  - [Dependencies](#dependencies)
  - [Configuration Files](#configuration-files)
  - [Database Setup](#database-setup)
  - [Authentication](#authentication)
  - [API Routes](#api-routes)
- [Frontend Setup](#frontend-setup)
  - [Dependencies](#dependencies-1)
  - [Configuration Files](#configuration-files-1)
  - [Components](#components)
  - [Authentication](#authentication-1)
- [Getting Started](#getting-started)
- [Deployment](#deployment)

## Project Structure

```
project-root/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── models.py
│   │   ├── database.py
│   │   ├── auth.py
│   │   └── routers/
│   │       └── users.py
│   ├── requirements.txt
│   └── alembic/
├── frontend/
│   ├── public/
│   ├── src/
│   │   ├── components/
│   │   │   └── Navbar.tsx
│   │   ├── pages/
│   │   │   └── Home.tsx
│   │   ├── App.tsx
│   │   ├── index.tsx
│   │   └── styles/
│   │       └── index.css
│   ├── tailwind.config.js
│   ├── package.json
│   └── tsconfig.json
├── docker-compose.yml
└── README.md
```

## Backend Setup

### Dependencies

Create a `requirements.txt` file to manage backend dependencies.

```typescript:backend/requirements.txt
fastapi
uvicorn
sqlalchemy
asyncpg
python-dotenv
authlib
python-jose
alembic
```

### Configuration Files

#### `.env`

Create a `.env` file in the `backend/` directory to store environment variables.

```env:backend/.env
DATABASE_URL=postgresql+asyncpg://user:password@db:5432/mydatabase
SECRET_KEY=your_secret_key
ALGORITHM=HS256
OAUTH_CLIENT_ID=your_oauth_client_id
OAUTH_CLIENT_SECRET=your_oauth_client_secret
```

### Database Setup

#### `database.py`

Handles the database connection using SQLAlchemy.

```typescript:backend/app/database.py
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
import os
from dotenv import load_dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")

engine: AsyncEngine = create_async_engine(DATABASE_URL, echo=True)
SessionLocal = sessionmaker(
    bind=engine, class_=AsyncSession, expire_on_commit=False
)

async def get_db():
    async with SessionLocal() as session:
        yield session
```

#### `models.py`

Defines the SQLAlchemy models.

```typescript:backend/app/models.py
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
```

### Authentication

#### `auth.py`

Manages OAuth authentication.

```typescript:backend/app/auth.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from datetime import datetime, timedelta
from .database import get_db
from sqlalchemy.ext.asyncio import AsyncSession
from . import models

import os
from dotenv import load_dotenv

load_dotenv()

SECRET_KEY = os.getenv("SECRET_KEY")
ALGORITHM = os.getenv("ALGORITHM")

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    user = await db.get(models.User, user_id)
    if user is None:
        raise credentials_exception
    return user
```

### API Routes

#### `users.py`

Defines user-related API endpoints.

```typescript:backend/app/routers/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from .. import models, database, auth
from typing import List

router = APIRouter(
    prefix="/users",
    tags=["users"],
)

@router.get("/", response_model=List[str])
async def read_users(db: AsyncSession = Depends(database.get_db), current_user: models.User = Depends(auth.get_current_user)):
    result = await db.execute(models.User.__table__.select())
    users = result.fetchall()
    return users
```

#### `main.py`

Entry point of the FastAPI application.

```typescript:backend/app/main.py
from fastapi import FastAPI
from .routers import users
from .database import engine
from . import models

app = FastAPI()

models.Base.metadata.create_all(bind=engine)

app.include_router(users.router)
```

## Frontend Setup

### Dependencies

Initialize the React project with TypeScript and install necessary dependencies.

```bash
cd frontend
npx create-react-app . --template typescript
npm install tailwindcss@latest postcss@latest autoprefixer@latest
npm install @shadcn/ui
npm install axios react-router-dom
```

### Configuration Files

#### `tailwind.config.js`

Configure Tailwind CSS.

```typescript:frontend/tailwind.config.js
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
    "./shadcn/**/*.{js,jsx,ts,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

#### `index.css`

Import Tailwind directives.

```typescript:frontend/src/styles/index.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### Components

#### `Navbar.tsx`

Sample Navbar component using Shadcn.

```typescript:frontend/src/components/Navbar.tsx
import React from 'react';
import { useAuth } from '../hooks/useAuth';

const Navbar: React.FC = () => {
    const { user, logout } = useAuth();

    return (
        <nav className="bg-gray-800 p-4">
            <div className="container mx-auto flex justify-between">
                <h1 className="text-white">MyApp</h1>
                {user ? (
                    <div className="flex items-center">
                        <span className="text-white mr-4">{user.email}</span>
                        <button onClick={logout} className="text-white">Logout</button>
                    </div>
                ) : (
                    <a href="/login" className="text-white">Login</a>
                )}
            </div>
        </nav>
    );
};

export default Navbar;
```

#### `App.tsx`

Main application component.

```typescript:frontend/src/App.tsx
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Navbar from './components/Navbar';
import Home from './pages/Home';
import Login from './pages/Login';

const App: React.FC = () => {
    return (
        <Router>
            <Navbar />
            <div className="container mx-auto mt-4">
                <Routes>
                    <Route path="/" element={<Home />} />
                    <Route path="/login" element={<Login />} />
                </Routes>
            </div>
        </Router>
    );
};

export default App;
```

### Authentication

#### `useAuth.ts`

Custom hook for authentication.

```typescript:frontend/src/hooks/useAuth.ts
import { useState, useEffect } from 'react';
import axios from 'axios';

interface User {
    id: number;
    email: string;
}

export const useAuth = () => {
    const [user, setUser] = useState<User | null>(null);

    useEffect(() => {
        const token = localStorage.getItem('token');
        if (token) {
            axios.get('/api/users/', {
                headers: {
                    Authorization: `Bearer ${token}`
                }
            })
            .then(response => {
                setUser(response.data);
            })
            .catch(() => {
                setUser(null);
            });
        }
    }, []);

    const login = (token: string) => {
        localStorage.setItem('token', token);
        // Fetch user data after login
    };

    const logout = () => {
        localStorage.removeItem('token');
        setUser(null);
    };

    return { user, login, logout };
};
```

## Getting Started

### Prerequisites

- **Docker** & **Docker Compose** installed
- **Node.js** and **npm** installed

### Setup Instructions

1. **Clone the Repository**

   ```bash
   git clone https://github.com/yourusername/project-template.git
   cd project-template
   ```

2. **Configure Environment Variables**

   - Backend: Edit `backend/.env` with your configurations.
   - Frontend: Update any necessary environment variables as needed.

3. **Run Docker Compose**

   ```bash
   docker-compose up --build
   ```

4. **Install Frontend Dependencies**

   ```bash
   cd frontend
   npm install
   npm run build
   ```

5. **Start Backend Server**

   The backend server will run automatically via Docker Compose on `http://localhost:8000`.

6. **Start Frontend Server**

   In another terminal:

   ```bash
   cd frontend
   npm start
   ```

   The frontend will be available at `http://localhost:3000`.

## Deployment

For deployment, consider using platforms like **Heroku**, **Vercel**, or **AWS**. Ensure environment variables are set appropriately in the production environment and update any deployment scripts as necessary.

### Docker Compose for Production

Ensure that `docker-compose.yml` is optimized for production with appropriate settings for security and performance.

```yaml:docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14
    restart: always
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydatabase
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build: ./backend
    restart: always
    env_file:
      - ./backend/.env
    depends_on:
      - db
    ports:
      - "8000:8000"

  frontend:
    build: ./frontend
    restart: always
    ports:
      - "3000:3000"

volumes:
  postgres_data:
```

---

This template sets up a scalable and maintainable full-stack application structure, leveraging modern technologies and best practices to accelerate development and deployment.