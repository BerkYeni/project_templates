# Project Template: FastAPI Backend with PostgreSQL and React Frontend

This template provides a full-stack setup using FastAPI for the backend, PostgreSQL for the database, and React with TypeScript, Tailwind CSS, and Shadcn for the frontend. It also includes OAuth authentication to secure your application.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Backend Setup](#backend-setup)
   - [Environment Configuration](#environment-configuration)
   - [Database Configuration](#database-configuration)
   - [Models](#models)
   - [Schemas](#schemas)
   - [Authentication](#authentication)
   - [Main Application](#main-application)
3. [Frontend Setup](#frontend-setup)
   - [Project Initialization](#project-initialization)
   - [Tailwind CSS Configuration](#tailwind-css-configuration)
   - [Shadcn Components](#shadcn-components)
   - [Authentication](#frontend-authentication)
   - [Main Application](#frontend-main-application)
4. [Running the Project](#running-the-project)

---

## Project Structure

```plaintext
project-root/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── database.py
│   │   └── auth.py
│   ├── alembic/
│   ├── requirements.txt
│   └── .env
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── App.tsx
│   │   ├── index.tsx
│   │   └── auth.tsx
│   ├── public/
│   ├── tailwind.config.js
│   ├── tsconfig.json
│   └── package.json
├── docker-compose.yml
└── README.md
```

---

## Backend Setup

### Environment Configuration

Create a `.env` file in the `backend/` directory to store environment variables.

```env
# backend/.env
DATABASE_URL=postgresql://user:password@db:5432/mydatabase
SECRET_KEY=your_secret_key
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
OAUTH_CLIENT_ID=your_oauth_client_id
OAUTH_CLIENT_SECRET=your_oauth_client_secret
```

### Database Configuration

Set up the database connection using SQLAlchemy.

```python:backend/app/database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os
from dotenv import load_dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
```

### Models

Define your database models.

```python:backend/app/models.py
from sqlalchemy import Column, Integer, String
from .database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String, nullable=True)
```

### Schemas

Create Pydantic schemas for data validation.

```python:backend/app/schemas.py
from pydantic import BaseModel, EmailStr
from typing import Optional

class UserBase(BaseModel):
    email: EmailStr
    full_name: Optional[str] = None

class UserCreate(UserBase):
    password: str

class UserRead(UserBase):
    id: int

    class Config:
        orm_mode = True
```

### Authentication

Implement OAuth authentication.

```python:backend/app/auth.py
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from . import models, schemas, database
from sqlalchemy.orm import Session
import os
from dotenv import load_dotenv

load_dotenv()

SECRET_KEY = os.getenv("SECRET_KEY")
ALGORITHM = os.getenv("ALGORITHM")
ACCESS_TOKEN_EXPIRE_MINUTES = int(os.getenv("ACCESS_TOKEN_EXPIRE_MINUTES"))

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def get_password_hash(password):
    return pwd_context.hash(password)

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def get_user(db: Session, email: str):
    return db.query(models.User).filter(models.User.email == email).first()

def authenticate_user(db: Session, email: str, password: str):
    user = get_user(db, email)
    if not user:
        return False
    if not verify_password(password, user.hashed_password):
        return False
    return user

async def get_current_user(token: str = Depends(oauth2_scheme), db: Session = Depends(database.SessionLocal)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        email: str = payload.get("sub")
        if email is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    user = get_user(db, email=email)
    if user is None:
        raise credentials_exception
    return user
```

### Main Application

Set up the FastAPI application and include routes.

```python:backend/app/main.py
from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from . import models, schemas, auth, database

models.Base.metadata.create_all(bind=database.engine)

app = FastAPI()

@app.post("/token", response_model=dict)
def login_for_access_token(form_data: auth.OAuth2PasswordRequestForm = Depends(), db: Session = Depends(database.SessionLocal)):
    user = auth.authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token_expires = timedelta(minutes=auth.ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = auth.create_access_token(
        data={"sub": user.email}, expires_delta=access_token_expires
    )
    return {"access_token": access_token, "token_type": "bearer"}

@app.post("/users/", response_model=schemas.UserRead)
def create_user(user: schemas.UserCreate, db: Session = Depends(database.SessionLocal)):
    db_user = auth.get_user(db, email=user.email)
    if db_user:
        raise HTTPException(status_code=400, detail="Email already registered")
    hashed_password = auth.get_password_hash(user.password)
    new_user = models.User(email=user.email, hashed_password=hashed_password, full_name=user.full_name)
    db.add(new_user)
    db.commit()
    db.refresh(new_user)
    return new_user

@app.get("/users/me/", response_model=schemas.UserRead)
def read_users_me(current_user: schemas.UserRead = Depends(auth.get_current_user)):
    return current_user
```

---

## Frontend Setup

### Project Initialization

Initialize a React project with TypeScript.

```bash
# Navigate to frontend directory
cd frontend

# Initialize React with TypeScript
npx create-react-app . --template typescript

# Install Tailwind CSS and dependencies
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# Install Shadcn UI
npm install shadcn-ui

# Install additional dependencies
npm install axios react-router-dom @types/react-router-dom
```

### Tailwind CSS Configuration

Configure Tailwind CSS by updating `tailwind.config.js`.

```javascript:frontend/tailwind.config.js
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
    "./public/index.html",
    "node_modules/shadcn-ui/**/*.{js,jsx,ts,tsx}"
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

Add Tailwind directives to `src/index.css`.

```css:frontend/src/index.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### Shadcn Components

Set up Shadcn UI components.

```typescript:frontend/src/components/Navbar.tsx
import React from 'react';
import { useAuth } from '../auth';

const Navbar: React.FC = () => {
  const { user, logout } = useAuth();

  return (
    <nav className="bg-gray-800 p-4">
      <div className="container mx-auto flex justify-between">
        <h1 className="text-white text-xl">MyApp</h1>
        <div>
          {user ? (
            <>
              <span className="text-white mr-4">Hello, {user.full_name}</span>
              <button onClick={logout} className="text-white">Logout</button>
            </>
          ) : (
            <a href="/login" className="text-white">Login</a>
          )}
        </div>
      </div>
    </nav>
  );
}

export default Navbar;
```

### Authentication

Create authentication context.

```typescript:frontend/src/auth.tsx
import React, { createContext, useContext, useState, useEffect } from 'react';
import axios from 'axios';

interface User {
  id: number;
  email: string;
  full_name?: string;
}

interface AuthContextType {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);

  const login = async (email: string, password: string) => {
    const response = await axios.post('/token', { username: email, password });
    localStorage.setItem('token', response.data.access_token);
    // Fetch user data
    const userResponse = await axios.get('/users/me/', {
      headers: {
        Authorization: `Bearer ${response.data.access_token}`
      }
    });
    setUser(userResponse.data);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.get('/users/me/', {
        headers: {
          Authorization: `Bearer ${token}`
        }
      })
      .then(response => setUser(response.data))
      .catch(() => logout());
    }
  }, []);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error("useAuth must be used within AuthProvider");
  return context;
}
```

### Main Application

Set up the main application with routing and authentication.

```typescript:frontend/src/App.tsx
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Navbar from './components/Navbar';
import Home from './pages/Home';
import Login from './pages/Login';
import { AuthProvider } from './auth';

const App: React.FC = () => {
  return (
    <AuthProvider>
      <Router>
        <Navbar />
        <div className="container mx-auto p-4">
          <Routes>
            <Route path="/" element={<Home />} />
            <Route path="/login" element={<Login />} />
          </Routes>
        </div>
      </Router>
    </AuthProvider>
  );
}

export default App;
```

```typescript:frontend/src/pages/Home.tsx
import React from 'react';

const Home: React.FC = () => {
  return (
    <div>
      <h2 className="text-2xl font-bold">Welcome to MyApp</h2>
      <p className="mt-4">This is the home page.</p>
    </div>
  );
}

export default Home;
```

```typescript:frontend/src/pages/Login.tsx
import React, { useState } from 'react';
import { useAuth } from '../auth';
import { useNavigate } from 'react-router-dom';

const Login: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const { login } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await login(email, password);
      navigate('/');
    } catch (error) {
      alert("Failed to login");
    }
  }

  return (
    <form onSubmit={handleSubmit} className="max-w-md mx-auto">
      <h2 className="text-xl mb-4">Login</h2>
      <div className="mb-4">
        <label className="block mb-1">Email</label>
        <input 
          type="email" 
          value={email} 
          onChange={e => setEmail(e.target.value)} 
          className="w-full border px-3 py-2"
          required 
        />
      </div>
      <div className="mb-4">
        <label className="block mb-1">Password</label>
        <input 
          type="password" 
          value={password} 
          onChange={e => setPassword(e.target.value)} 
          className="w-full border px-3 py-2"
          required 
        />
      </div>
      <button type="submit" className="w-full bg-blue-500 text-white py-2">Login</button>
    </form>
  );
}

export default Login;
```

---

## Running the Project

### Docker Setup

Use Docker Compose to manage backend and PostgreSQL services.

```yaml:docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:13
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
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    env_file:
      - ./backend/.env
    depends_on:
      - db

  frontend:
    build: ./frontend
    command: npm start
    volumes:
      - ./frontend:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    depends_on:
      - backend

volumes:
  postgres_data:
```

### Backend Dockerfile

Create a `Dockerfile` for the backend.

```dockerfile:backend/Dockerfile
FROM python:3.9

WORKDIR /app

COPY requirements.txt .

RUN pip install --upgrade pip
RUN pip install -r requirements.txt

COPY . .

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### Frontend Dockerfile

Create a `Dockerfile` for the frontend.

```dockerfile:frontend/Dockerfile
FROM node:16

WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install

COPY . .

CMD ["yarn", "start"]
```

### Requirements

List backend dependencies in `backend/requirements.txt`.

```plaintext:backend/requirements.txt
fastapi
uvicorn
sqlalchemy
psycopg2-binary
python-dotenv
jose
passlib[bcrypt]
```

### Running the Application

Start the services using Docker Compose.

```bash
docker-compose up --build
```

Access the frontend at `http://localhost:3000` and the backend API at `http://localhost:8000`.

---

## Next Steps

- **Error Handling**: Implement comprehensive error handling on both frontend and backend.
- **Testing**: Add unit and integration tests.
- **Deployment**: Configure CI/CD pipelines for deployment.
- **Enhancements**: Expand authentication methods, implement role-based access, and add more features as needed.

This template serves as a starting point. Customize it according to your project requirements.