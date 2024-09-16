# Project Template: Go Backend with PostgreSQL and Modern Frontend

This project template provides a robust foundation for developers to quickly build and deploy web applications. It leverages Go for the backend, PostgreSQL for the database, and utilizes HTMX, Tailwind CSS, and Shadcn for the frontend. OAuth is implemented for secure authentication.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Backend Setup](#backend-setup)
   - [main.go](#main.go)
   - [Configuration](#configuration)
   - [Database Connection](#database-connection)
   - [Models](#models)
   - [Handlers](#handlers)
   - [Routes](#routes)
   - [Middleware](#middleware)
3. [Frontend Setup](#frontend-setup)
   - [index.html](#index.html)
   - [Tailwind Configuration](#tailwind-configuration)
4. [Authentication with OAuth](#authentication-with-oauth)
5. [Running the Project](#running-the-project)
6. [Dependencies](#dependencies)

---

## Project Structure

```
myproject/
├── backend/
│   ├── config/
│   │   └── config.go
│   ├── controllers/
│   │   └── userController.go
│   ├── models/
│   │   └── user.go
│   ├── routes/
│   │   └── routes.go
│   ├── middleware/
│   │   └── authMiddleware.go
│   └── main.go
├── frontend/
│   ├── public/
│   │   └── index.html
│   ├── styles/
│   │   └── tailwind.css
│   └── components/
│       └── Navbar.tsx
├── .env
├── go.mod
└── README.md
```

---

## Backend Setup

### `backend/main.go`

```go:backend/main.go
package main

import (
    "log"
    "myproject/backend/config"
    "myproject/backend/routes"
)

func main() {
    // Initialize configuration
    config.Init()

    // Setup routes
    router := routes.SetupRouter()

    // Start the server
    log.Println("Server running on port 8080")
    if err := router.Run(":8080"); err != nil {
        log.Fatalf("Could not run server: %v", err)
    }
}
```

### Configuration

#### `backend/config/config.go`

```go:backend/config/config.go
package config

import (
    "log"
    "os"

    "github.com/joho/godotenv"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

var DB *gorm.DB

func Init() {
    // Load environment variables
    err := godotenv.Load()
    if err != nil {
        log.Fatalf("Error loading .env file")
    }

    // Database connection
    dsn := os.Getenv("DATABASE_URL")
    DB, err = gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }

    // Migrate models
    // Example: DB.AutoMigrate(&models.User{})
}
```

### Database Connection

The database connection is handled in the `config.go` file using GORM. Ensure your `.env` file contains the `DATABASE_URL`.

### Models

#### `backend/models/user.go`

```go:backend/models/user.go
package models

import "gorm.io/gorm"

type User struct {
    gorm.Model
    Name     string `gorm:"not null"`
    Email    string `gorm:"uniqueIndex;not null"`
    Password string `gorm:"not null"`
}
```

### Handlers

#### `backend/controllers/userController.go`

```go:backend/controllers/userController.go
package controllers

import (
    "net/http"
    "myproject/backend/models"

    "github.com/gin-gonic/gin"
)

func GetUsers(c *gin.Context) {
    var users []models.User
    models.DB.Find(&users)
    c.JSON(http.StatusOK, users)
}

func CreateUser(c *gin.Context) {
    var input models.User
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    models.DB.Create(&input)
    c.JSON(http.StatusOK, input)
}
```

### Routes

#### `backend/routes/routes.go`

```go:backend/routes/routes.go
package routes

import (
    "myproject/backend/controllers"
    "myproject/backend/middleware"

    "github.com/gin-gonic/gin"
)

func SetupRouter() *gin.Engine {
    router := gin.Default()

    // Public routes
    router.POST("/login", controllers.Login)
    router.POST("/signup", controllers.Signup)

    // Protected routes
    protected := router.Group("/api")
    protected.Use(middleware.AuthMiddleware())
    {
        protected.GET("/users", controllers.GetUsers)
        protected.POST("/users", controllers.CreateUser)
    }

    return router
}
```

### Middleware

#### `backend/middleware/authMiddleware.go`

```go:backend/middleware/authMiddleware.go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
)

func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Authorization header required"})
            return
        }

        token := strings.TrimPrefix(authHeader, "Bearer ")
        if token != "your-oauth-token" { // Replace with actual token validation
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
            return
        }

        c.Next()
    }
}
```

---

## Frontend Setup

### `frontend/public/index.html`

```html:frontend/public/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>MyProject</title>
    <link href="/styles/tailwind.css" rel="stylesheet">
    <script src="https://unpkg.com/htmx.org@1.9.2"></script>
</head>
<body class="bg-gray-100">
    <nav class="bg-white shadow">
        <div class="max-w-7xl mx-auto px-4">
            <div class="flex justify-between">
                <div class="flex space-x-4">
                    <a href="/" class="flex items-center py-5 px-2 text-gray-700">Home</a>
                    <a href="/about" class="flex items-center py-5 px-2 text-gray-700">About</a>
                </div>
                <div class="flex items-center space-x-4">
                    <button class="px-3 py-2 bg-blue-500 text-white rounded">Login</button>
                </div>
            </div>
        </div>
    </nav>

    <div class="container mx-auto mt-5">
        <!-- Content goes here -->
    </div>

    <script src="/components/Navbar.tsx"></script>
</body>
</html>
```

### Tailwind Configuration

#### `frontend/styles/tailwind.css`

```css:frontend/styles/tailwind.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom styles */
```

**Tailwind Configuration File (`tailwind.config.js`)**

```javascript:tailwind.config.js
module.exports = {
  purge: ['./frontend/public/**/*.html', './frontend/components/**/*.tsx'],
  darkMode: false, // or 'media' or 'class'
  theme: {
    extend: {},
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}
```

---

## Authentication with OAuth

OAuth authentication is implemented using [golang/oauth2](https://github.com/golang/oauth2) package. Below is an example setup using GitHub as the OAuth provider.

#### `backend/controllers/authController.go`

```go:backend/controllers/authController.go
package controllers

import (
    "context"
    "fmt"
    "net/http"
    "os"

    "github.com/gin-gonic/gin"
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/github"
)

var oauthConf = &oauth2.Config{
    RedirectURL:  "http://localhost:8080/auth/callback",
    ClientID:     os.Getenv("GITHUB_CLIENT_ID"),
    ClientSecret: os.Getenv("GITHUB_CLIENT_SECRET"),
    Scopes:       []string{"user:email"},
    Endpoint:     github.Endpoint,
}

func Login(c *gin.Context) {
    url := oauthConf.AuthCodeURL("state", oauth2.AccessTypeOffline)
    c.Redirect(http.StatusTemporaryRedirect, url)
}

func AuthCallback(c *gin.Context) {
    code := c.Query("code")
    token, err := oauthConf.Exchange(context.Background(), code)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Failed to exchange token"})
        return
    }

    // Use token to get user info
    client := oauthConf.Client(context.Background(), token)
    resp, err := client.Get("https://api.github.com/user")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Failed to get user info"})
        return
    }
    defer resp.Body.Close()

    // Handle user info and create session
    // ...

    c.JSON(http.StatusOK, gin.H{"message": "Authentication successful"})
}
```

---

## Running the Project

### Prerequisites

- **Go**: Install [Go](https://golang.org/dl/)
- **Node.js & npm**: Install [Node.js](https://nodejs.org/)
- **PostgreSQL**: Install [PostgreSQL](https://www.postgresql.org/download/)

### Setup Steps

1. **Clone the Repository**

    ```bash
    git clone https://github.com/yourusername/myproject.git
    cd myproject
    ```

2. **Backend Setup**

    - Navigate to the backend directory:

        ```bash
        cd backend
        ```

    - Create a `.env` file based on `.env.example`:

        ```env
        DATABASE_URL=postgres://user:password@localhost:5432/mydb?sslmode=disable
        GITHUB_CLIENT_ID=your_github_client_id
        GITHUB_CLIENT_SECRET=your_github_client_secret
        ```

    - Install dependencies:

        ```bash
        go mod download
        ```

    - Run database migrations:

        ```bash
        go run main.go
        ```

3. **Frontend Setup**

    - Navigate to the frontend directory:

        ```bash
        cd ../frontend
        ```

    - Initialize Tailwind CSS:

        ```bash
        npx tailwindcss init
        ```

    - Build Tailwind CSS:

        ```bash
        npx tailwindcss -i ./styles/tailwind.css -o ./styles/output.css --watch
        ```

4. **Run the Application**

    - Start the backend server:

        ```bash
        cd ../backend
        go run main.go
        ```

    - Access the frontend at `http://localhost:8080`

---

## Dependencies

### Backend

- [Gin Web Framework](https://github.com/gin-gonic/gin)
- [GORM ORM](https://gorm.io/)
- [godotenv](https://github.com/joho/godotenv)
- [golang/oauth2](https://github.com/golang/oauth2)

### Frontend

- [HTMX](https://htmx.org/)
- [Tailwind CSS](https://tailwindcss.com/)
- [Shadcn/UI](https://shadcn.com/)

---

## Conclusion

This template serves as a solid starting point for developing scalable web applications with modern technologies. Customize and expand upon this foundation to suit your project's specific needs.

# License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

# Contact

For any inquiries or feedback, please contact [your.email@example.com](mailto:your.email@example.com).

# Acknowledgments

- Inspired by best practices in Go and modern frontend development.
- Thanks to the open-source community for providing invaluable tools and libraries.

# Happy Coding! 🚀