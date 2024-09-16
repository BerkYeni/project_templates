# Go-PostgreSQL Backend with HTMX, Tailwind, Shadcn Frontend and OAuth Authentication

Below is a comprehensive backend project template that leverages **Go** for the backend, **PostgreSQL** for the database, and a frontend built with **HTMX**, **Tailwind CSS**, and **Shadcn UI**. Additionally, OAuth is integrated for authentication.

---

## Project Structure

```
my-go-project/
├── backend/
│   ├── cmd/
│   │   └── server/
│   │       └── main.go
│   ├── internal/
│   │   ├── config/
│   │   │   └── config.go
│   │   ├── controllers/
│   │   │   └── authController.go
│   │   ├── models/
│   │   │   └── user.go
│   │   ├── routes/
│   │   │   └── routes.go
│   │   ├── services/
│   │   │   └── authService.go
│   │   └── utils/
│   │       └── db.go
│   ├── .env
│   ├── go.mod
│   └── go.sum
├── frontend/
│   ├── public/
│   │   └── index.html
│   ├── src/
│   │   ├── components/
│   │   │   └── Navbar.tsx
│   │   ├── pages/
│   │   │   └── Home.tsx
│   │   ├── styles/
│   │   │   └── index.css
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   ├── package.json
│   └── tsconfig.json
├── docker-compose.yml
├── README.md
└── .gitignore
```

---

## Backend Setup

### 1. Initialize Backend

```bash
cd backend
go mod init github.com/your-username/my-go-project
go get -u github.com/gin-gonic/gin
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
go get -u github.com/joho/godotenv
go get -u golang.org/x/oauth2
go get -u golang.org/x/oauth2/google
```

### 2. `backend/go.mod`

```go:backend/go.mod
module github.com/your-username/my-go-project

go 1.20

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/joho/godotenv v1.4.0
    golang.org/x/oauth2 v0.0.0-20230810222323-740e4c2c4c78
    gorm.io/driver/postgres v1.4.6
    gorm.io/gorm v1.25.1
)
```

### 3. Environment Variables

Create a `.env` file in the `backend/` directory:

```
PORT=8080
DATABASE_URL=postgres://username:password@db:5432/mydatabase?sslmode=disable
OAUTH_CLIENT_ID=your-google-client-id
OAUTH_CLIENT_SECRET=your-google-client-secret
OAUTH_REDIRECT_URL=http://localhost:8080/auth/callback
SESSION_SECRET=your-session-secret
```

### 4. Database Utility

#### `backend/internal/utils/db.go`

```go:backend/internal/utils/db.go
package utils

import (
    "log"
    "os"

    "github.com/joho/godotenv"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

var DB *gorm.DB

func InitDB() {
    err := godotenv.Load()
    if err != nil {
        log.Fatalf("Error loading .env file: %v", err)
    }

    dsn := os.Getenv("DATABASE_URL")
    DB, err = gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }

    log.Println("Database connection established")
}
```

### 5. Configuration

#### `backend/internal/config/config.go`

```go:backend/internal/config/config.go
package config

import (
    "os"

    "github.com/joho/godotenv"
)

type Config struct {
    Port           string
    OAuthClientID     string
    OAuthClientSecret string
    OAuthRedirectURL  string
    SessionSecret     string
}

func LoadConfig() Config {
    err := godotenv.Load()
    if err != nil {
        panic("Error loading .env file")
    }

    return Config{
        Port:             os.Getenv("PORT"),
        OAuthClientID:    os.Getenv("OAUTH_CLIENT_ID"),
        OAuthClientSecret: os.Getenv("OAUTH_CLIENT_SECRET"),
        OAuthRedirectURL:  os.Getenv("OAUTH_REDIRECT_URL"),
        SessionSecret:     os.Getenv("SESSION_SECRET"),
    }
}
```

### 6. User Model

#### `backend/internal/models/user.go`

```go:backend/internal/models/user.go
package models

import (
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string `gorm:"not null"`
    Email string `gorm:"uniqueIndex;not null"`
}
```

### 7. Database Initialization

Update `backend/cmd/server/main.go` to include database migration.

#### `backend/cmd/server/main.go`

```go:backend/cmd/server/main.go
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/your-username/my-go-project/internal/config"
    "github.com/your-username/my-go-project/internal/models"
    "github.com/your-username/my-go-project/internal/routes"
    "github.com/your-username/my-go-project/internal/utils"
)

func main() {
    config := config.LoadConfig()
    utils.InitDB()

    // Migrate the schema
    utils.DB.AutoMigrate(&models.User{})

    router := gin.Default()

    routes.SetupRoutes(router)

    router.Run(":" + config.Port)
}
```

### 8. Routes Setup

#### `backend/internal/routes/routes.go`

```go:backend/internal/routes/routes.go
package routes

import (
    "github.com/gin-gonic/gin"
    "github.com/your-username/my-go-project/internal/controllers"
)

func SetupRoutes(router *gin.Engine) {
    router.GET("/", controllers.Home)
    router.GET("/auth/oauth", controllers.OAuthLogin)
    router.GET("/auth/callback", controllers.OAuthCallback)
    router.GET("/users", controllers.GetUsers)
}
```

### 9. Controllers

#### `backend/internal/controllers/authController.go`

```go:backend/internal/controllers/authController.go
package controllers

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"

    "github.com/gin-gonic/gin"
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
    "github.com/your-username/my-go-project/internal/config"
    "github.com/your-username/my-go-project/internal/models"
    "github.com/your-username/my-go-project/internal/utils"
)

var oauthConf *oauth2.Config

func init() {
    cfg := config.LoadConfig()
    oauthConf = &oauth2.Config{
        RedirectURL:  cfg.OAuthRedirectURL,
        ClientID:     cfg.OAuthClientID,
        ClientSecret: cfg.OAuthClientSecret,
        Scopes:       []string{"https://www.googleapis.com/auth/userinfo.email", "https://www.googleapis.com/auth/userinfo.profile"},
        Endpoint:     google.Endpoint,
    }
}

func OAuthLogin(c *gin.Context) {
    url := oauthConf.AuthCodeURL("state", oauth2.AccessTypeOffline)
    c.Redirect(http.StatusTemporaryRedirect, url)
}

func OAuthCallback(c *gin.Context) {
    code := c.Query("code")
    token, err := oauthConf.Exchange(context.Background(), code)
    if err != nil {
        log.Println("Error exchanging code for token:", err)
        c.Redirect(http.StatusTemporaryRedirect, "/")
        return
    }

    client := oauthConf.Client(context.Background(), token)
    resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
    if err != nil {
        log.Println("Error fetching user info:", err)
        c.Redirect(http.StatusTemporaryRedirect, "/")
        return
    }
    defer resp.Body.Close()

    var userInfo struct {
        ID    string `json:"id"`
        Email string `json:"email"`
        Name  string `json:"name"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&userInfo); err != nil {
        log.Println("Error decoding user info:", err)
        c.Redirect(http.StatusTemporaryRedirect, "/")
        return
    }

    var user models.User
    result := utils.DB.Where("email = ?", userInfo.Email).First(&user)
    if result.Error != nil {
        // User not found, create new
        user = models.User{
            Name:  userInfo.Name,
            Email: userInfo.Email,
        }
        utils.DB.Create(&user)
    }

    // Here you can set session or JWT
    // For simplicity, we'll just return user info
    c.JSON(http.StatusOK, gin.H{
        "message": "Authentication successful",
        "user":    user,
    })
}

func Home(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "message": "Welcome to the Go Backend!",
    })
}
```

#### `backend/internal/controllers/userController.go`

```go:backend/internal/controllers/userController.go
package controllers

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/your-username/my-go-project/internal/models"
    "github.com/your-username/my-go-project/internal/utils"
)

func GetUsers(c *gin.Context) {
    var users []models.User
    result := utils.DB.Find(&users)
    if result.Error != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to retrieve users"})
        return
    }
    c.JSON(http.StatusOK, users)
}
```

### 10. OAuth Service

For this template, OAuth logic is handled within the `authController.go`. If you prefer to separate concerns, you can create a dedicated service.

#### `backend/internal/services/authService.go`

```go:backend/internal/services/authService.go
package services

import (
    "context"
    "log"

    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

type AuthService struct {
    OAuthConf *oauth2.Config
}

func NewAuthService(clientID, clientSecret, redirectURL string) *AuthService {
    return &AuthService{
        OAuthConf: &oauth2.Config{
            RedirectURL:  redirectURL,
            ClientID:     clientID,
            ClientSecret: clientSecret,
            Scopes:       []string{"https://www.googleapis.com/auth/userinfo.email", "https://www.googleapis.com/auth/userinfo.profile"},
            Endpoint:     google.Endpoint,
        },
    }
}

func (a *AuthService) GetAuthURL(state string) string {
    return a.OAuthConf.AuthCodeURL(state, oauth2.AccessTypeOffline)
}

func (a *AuthService) ExchangeToken(code string) (*oauth2.Token, error) {
    return a.OAuthConf.Exchange(context.Background(), code)
}

func (a *AuthService) GetUserInfo(token *oauth2.Token) (*UserInfo, error) {
    client := a.OAuthConf.Client(context.Background(), token)
    resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var userInfo UserInfo
    if err := json.NewDecoder(resp.Body).Decode(&userInfo); err != nil {
        return nil, err
    }

    return &userInfo, nil
}

type UserInfo struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name"`
}

func (u *UserInfo) Validate() bool {
    return u.Email != "" && u.Name != ""
}
```

---

## Frontend Setup

### 1. Initialize Frontend

```bash
cd frontend
npm init vite@latest . -- --template react-ts
npm install
npm install htmx.org tailwindcss postcss autoprefixer @shadcn/ui
npx tailwindcss init -p
```

### 2. `frontend/tailwind.config.js`

```javascript:frontend/tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./index.html",
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

### 4. `frontend/src/styles/index.css`

```css:frontend/src/styles/index.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 5. `frontend/public/index.html`

```html:frontend/public/index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" href="/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Go-PostgreSQL Project</title>
    <script src="https://unpkg.com/htmx.org@1.9.2"></script>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### 6. `frontend/src/main.tsx`

```typescript:frontend/src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './styles/index.css';

ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

### 7. `frontend/src/App.tsx`

```typescript:frontend/src/App.tsx
import React from 'react';
import Home from './pages/Home';
import Dashboard from './pages/Dashboard';
import { Navbar } from './components/Navbar';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <Router>
      <Navbar />
      <div className="container mx-auto mt-4">
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </div>
    </Router>
  )
}

export default App;
```

### 8. Example Page Component

#### `frontend/src/pages/Home.tsx`

```typescript:frontend/src/pages/Home.tsx
import React from 'react';
import { Button } from '@shadcn/ui';

const Home: React.FC = () => {
    const handleLogin = () => {
        window.location.href = 'http://localhost:8080/auth/oauth';
    };

    return (
        <div className="flex items-center justify-center h-screen bg-gray-100">
            <Button onClick={handleLogin}>
                Login with OAuth
            </Button>
        </div>
    );
};

export default Home;
```

#### `frontend/src/pages/Dashboard.tsx`

```typescript:frontend/src/pages/Dashboard.tsx
import React, { useEffect, useState } from 'react';
import axios from 'axios';
import { User } from '../types/User';

const Dashboard: React.FC = () => {
    const [users, setUsers] = useState<User[]>([]);

    useEffect(() => {
        fetchUsers();
    }, []);

    const fetchUsers = async () => {
        try {
            const response = await axios.get('http://localhost:8080/users');
            setUsers(response.data);
        } catch (error) {
            console.error("Error fetching users:", error);
        }
    };

    return (
        <div>
            <h1 className="text-2xl font-bold mb-4">User Dashboard</h1>
            <table className="min-w-full bg-white">
                <thead>
                    <tr>
                        <th className="py-2">ID</th>
                        <th className="py-2">Name</th>
                        <th className="py-2">Email</th>
                    </tr>
                </thead>
                <tbody>
                    {users.map(user => (
                        <tr key={user.ID}>
                            <td className="border px-4 py-2">{user.ID}</td>
                            <td className="border px-4 py-2">{user.Name}</td>
                            <td className="border px-4 py-2">{user.Email}</td>
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
};

export default Dashboard;
```

### 9. Navbar Component

#### `frontend/src/components/Navbar.tsx`

```typescript:frontend/src/components/Navbar.tsx
import React from 'react';
import { Link } from 'react-router-dom';

export const Navbar: React.FC = () => {
    return (
        <nav className="bg-blue-500 p-4">
            <div className="container mx-auto flex justify-between">
                <Link to="/" className="text-white font-bold">Home</Link>
                <Link to="/dashboard" className="text-white">Dashboard</Link>
            </div>
        </nav>
    );
};
```

### 10. User Type Definition

#### `frontend/src/types/User.ts`

```typescript:frontend/src/types/User.ts
export interface User {
    ID: number;
    Name: string;
    Email: string;
}
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
      - "8080:8080"
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

### `backend/Dockerfile`

```dockerfile:backend/Dockerfile
# Use the official Golang image as the base
FROM golang:1.20-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy go.mod and go.sum files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy the rest of the application code
COPY . .

# Build the Go application
RUN go build -o server ./cmd/server

# Expose port 8080
EXPOSE 8080

# Command to run the executable
CMD ["./server"]
```

### `frontend/Dockerfile`

```dockerfile:frontend/Dockerfile
# Use the official Node.js image as the base
FROM node:18-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy package.json and package-lock.json
COPY package.json tsconfig.json tailwind.config.js postcss.config.js ./

# Install dependencies
RUN npm install

# Copy the rest of the application code
COPY ./src ./src
COPY ./public ./public

# Build the frontend
RUN npm run build

# Install serve to serve the build
RUN npm install -g serve

# Expose port 3000
EXPOSE 3000

# Command to serve the build
CMD ["serve", "-s", "dist", "-l", "3000"]
```

---

## Getting Started

1. **Clone the Repository**

    ```bash
    git clone https://github.com/your-username/my-go-project.git
    cd my-go-project
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
    - **Backend API:** [http://localhost:8080](http://localhost:8080)

---

## Technologies Used

- **Backend:**
  - Go
  - Gin Web Framework
  - GORM
  - PostgreSQL
  - OAuth2 (Google)
  - Godotenv

- **Frontend:**
  - React
  - TypeScript
  - HTMX
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