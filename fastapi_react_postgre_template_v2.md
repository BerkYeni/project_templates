# Project Template: FastAPI Backend with React Frontend

This project template provides a scalable and maintainable structure to quickly build and deploy applications using FastAPI for the backend, PostgreSQL for the database, and React with TypeScript, Tailwind CSS, and ShadCN for the frontend. OAuth is integrated for authentication.

## Table of Contents

- [Project Structure](#project-structure)
- [Backend Setup](#backend-setup)
  - [Dependencies](#dependencies)
  - [Configuration](#configuration)
  - [Database Models](#database-models)
  - [Authentication](#authentication)
  - [API Routes](#api-routes)
- [Frontend Setup](#frontend-setup)
  - [Dependencies](#dependencies-1)
  - [Configuration](#configuration-1)
  - [Components](#components)
  - [Authentication](#authentication-1)
- [Deployment](#deployment)
- [Scripts](#scripts)
- [Environment Variables](#environment-variables)

## Project Structure

```plaintext
project-root/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── crud.py
│   │   ├── database.py
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   ├── oauth.py
│   ├── requirements.txt
│   └── alembic/
│       └── ... # Alembic migration files
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Header.tsx
│   │   │   ├── Footer.tsx
│   │   ├── pages/
│   │   │   ├── Home.tsx
│   │   │   ├── Login.tsx
│   │   ├── App.tsx
│   │   ├── index.tsx
│   ├── tailwind.config.js
│   ├── tsconfig.json
│   └── package.json
├── docker-compose.yml
├── README.md
└── .env
```

## Backend Setup

### 1. Dependencies

Create a `requirements.txt` file in the `backend/` directory:

```plaintext:backend/requirements.txt
fastapi
uvicorn
SQLAlchemy
psycopg2-binary
alembic
python-dotenv
oauthlib
python-jose
```

### 2. Configuration

Create a `.env` file in the root directory:

```dotenv:.env
DATABASE_URL=postgresql://user:password@localhost:5432/mydatabase
SECRET_KEY=your_secret_key
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
```

### 3. Database Setup

#### `database.py`

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

#### `models.py`

```python:backend/app/models.py
from sqlalchemy import Column, Integer, String
from .database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)
```

#### `schemas.py`

```python:backend/app/schemas.py
from pydantic import BaseModel

class UserBase(BaseModel):
    username: str
    email: str

class UserCreate(UserBase):
    password: str

class User(UserBase):
    id: int

    class Config:
        orm_mode = True
```

#### `crud.py`

```python:backend/app/crud.py
from sqlalchemy.orm import Session
from . import models, schemas
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def get_user(db: Session, user_id: int):
    return db.query(models.User).filter(models.User.id == user_id).first()

def get_user_by_email(db: Session, email: str):
    return db.query(models.User).filter(models.User.email == email).first()

def create_user(db: Session, user: schemas.UserCreate):
    hashed_password = pwd_context.hash(user.password)
    db_user = models.User(username=user.username, email=user.email, hashed_password=hashed_password)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

### 4. Authentication

#### `auth/oauth.py`

```python:backend/app/auth/oauth.py
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session
from fastapi.security import OAuth2PasswordBearer
from .. import schemas, crud, database

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

SECRET_KEY = os.getenv("SECRET_KEY")
ALGORITHM = os.getenv("ALGORITHM")
ACCESS_TOKEN_EXPIRE_MINUTES = int(os.getenv("ACCESS_TOKEN_EXPIRE_MINUTES"))

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta if expires_delta else timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def get_current_user(token: str = Depends(oauth2_scheme), db: Session = Depends(database.SessionLocal)):
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
    user = crud.get_user(db, user_id=user_id)
    if user is None:
        raise credentials_exception
    return user
```

### 5. API Routes

#### `main.py`

```python:backend/app/main.py
from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from . import models, schemas, crud, database, auth.oauth
from fastapi.security import OAuth2PasswordRequestForm

models.Base.metadata.create_all(bind=database.engine)

app = FastAPI()

@app.post("/users/", response_model=schemas.User)
def create_user(user: schemas.UserCreate, db: Session = Depends(database.SessionLocal)):
    db_user = crud.get_user_by_email(db, email=user.email)
    if db_user:
        raise HTTPException(status_code=400, detail="Email already registered")
    return crud.create_user(db=db, user=user)

@app.post("/token", response_model=schemas.Token)
def login_for_access_token(form_data: OAuth2PasswordRequestForm = Depends(), db: Session = Depends(database.SessionLocal)):
    user = crud.get_user_by_email(db, email=form_data.username)
    if not user or not auth.oauth.pwd_context.verify(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token = auth.oauth.create_access_token(data={"sub": user.id})
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/users/me/", response_model=schemas.User)
def read_users_me(current_user: schemas.User = Depends(auth.oauth.get_current_user)):
    return current_user
```

## Frontend Setup

### 1. Dependencies

Initialize the frontend with React, TypeScript, Tailwind CSS, and ShadCN:

```bash
npx create-react-app frontend --template typescript
cd frontend
npm install tailwindcss shadcn
```

### 2. Configuration

#### `tailwind.config.js`

```javascript:frontend/tailwind.config.js
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
    "./public/index.html",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

#### `tsconfig.json`

Ensure TypeScript is properly configured:

```json:frontend/tsconfig.json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx"
  },
  "include": ["src"]
}
```

### 3. Components

#### `Header.tsx`

```typescript:frontend/src/components/Header.tsx
import React from 'react';

const Header: React.FC = () => {
  return (
    <header className="bg-blue-500 p-4 text-white">
      <h1>My App</h1>
    </header>
  );
};

export default Header;
```

#### `Footer.tsx`

```typescript:frontend/src/components/Footer.tsx
import React from 'react';

const Footer: React.FC = () => {
  return (
    <footer className="bg-gray-800 p-4 text-white">
      <p>&copy; 2023 My App</p>
    </footer>
  );
};

export default Footer;
```

### 4. Pages

#### `Home.tsx`

```typescript:frontend/src/pages/Home.tsx
import React from 'react';

const Home: React.FC = () => {
  return (
    <div className="container mx-auto">
      <h2 className="text-2xl">Welcome to the Home Page</h2>
    </div>
  );
};

export default Home;
```

#### `Login.tsx`

```typescript:frontend/src/pages/Login.tsx
import React, { useState } from 'react';
import axios from 'axios';

const Login: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      const response = await axios.post('/token', {
        username: email,
        password: password,
      });
      localStorage.setItem('token', response.data.access_token);
      // Redirect or update UI
    } catch (error) {
      console.error("Login failed", error);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="max-w-md mx-auto">
      <div>
        <label>Email:</label>
        <input type="email" value={email} onChange={e => setEmail(e.target.value)} required className="border p-2 w-full"/>
      </div>
      <div>
        <label>Password:</label>
        <input type="password" value={password} onChange={e => setPassword(e.target.value)} required className="border p-2 w-full"/>
      </div>
      <button type="submit" className="bg-blue-500 text-white p-2 mt-4">Login</button>
    </form>
  );
};

export default Login;
```

### 5. Main App

#### `App.tsx`

```typescript:frontend/src/App.tsx
import React from 'react';
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom';
import Header from './components/Header';
import Footer from './components/Footer';
import Home from './pages/Home';
import Login from './pages/Login';

const App: React.FC = () => {
  return (
    <Router>
      <Header />
      <main className="flex-grow">
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/login" element={<Login />} />
        </Routes>
      </main>
      <Footer />
    </Router>
  );
};

export default App;
```

#### `index.tsx`

```typescript:frontend/src/index.tsx
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
);
```

## Deployment

Use Docker and Docker Compose to containerize the application.

### `docker-compose.yml`

```yaml:docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:13
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
    env_file:
      - .env
    ports:
      - "8000:8000"
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

## Scripts

### Backend `Dockerfile`

Create a `Dockerfile` in the `backend/` directory:

```dockerfile:backend/Dockerfile
FROM python:3.9

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Frontend `Dockerfile`

Create a `Dockerfile` in the `frontend/` directory:

```dockerfile:frontend/Dockerfile
FROM node:16

WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install

COPY . .

CMD ["yarn", "start"]
```

## Environment Variables

Ensure that the `.env` file contains all necessary configurations for both backend and frontend. Sensitive information like `SECRET_KEY` should be kept secure and not committed to version control.

```dotenv:.env
# Backend
DATABASE_URL=postgresql://user:password@db:5432/mydatabase
SECRET_KEY=your_secret_key
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# Frontend
REACT_APP_API_URL=http://localhost:8000
```

---

This template sets up a full-stack application with a FastAPI backend and a React frontend, integrating PostgreSQL for data persistence and OAuth for authentication. Tailwind CSS and ShadCN ensure a modern and responsive UI. Docker and Docker Compose facilitate easy deployment and environment setup.

Feel free to customize and expand upon this foundation to suit your project's specific needs!