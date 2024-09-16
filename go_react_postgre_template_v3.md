# Project Template: Go Backend with PostgreSQL, HTMX, Tailwind, Shadcn, and OAuth Authentication

This project template provides a robust foundation for building web applications with a Go backend, PostgreSQL database, and a modern frontend stack using HTMX, Tailwind CSS, and Shadcn components. It also includes OAuth authentication to secure your application.

## Directory Structure

```
project-root/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── auth/
│   │   └── oauth.go
│   ├── handlers/
│   │   ├── home.go
│   │   └── user.go
│   ├── models/
│   │   └── user.go
│   └── db/
│       └── db.go
├── web/
│   ├── static/
│   │   ├── css/
│   │   │   ├── tailwind.css
│   │   │   └── output.css
│   │   └── js/
│   │       └── main.js
│   ├── templates/
│   │   ├── base.html
│   │   ├── home.html
│   │   └── login.html
│   └── components/
│       └── navbar.html
├── migrations/
│   └── 001_create_users_table.sql
├── .env
├── Dockerfile
├── docker-compose.yml
├── go.mod
└── go.sum
```

## Setup Instructions

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/yourusername/project-template.git
   cd project-template
   ```

2. **Configure Environment Variables:**

   Create a `.env` file in the root directory:

   ```env
   DATABASE_URL=postgres://user:password@localhost:5432/yourdb?sslmode=disable
   OAUTH_CLIENT_ID=your-client-id
   OAUTH_CLIENT_SECRET=your-client-secret
   OAUTH_REDIRECT_URL=http://localhost:8080/auth/callback
   SESSION_SECRET=your-session-secret
   PORT=8080
   ```

3. **Run Database Migrations:**

   Apply the SQL migration to set up the database:

   ```bash
   psql $DATABASE_URL -f migrations/001_create_users_table.sql
   ```

4. **Install Dependencies:**

   Ensure you have Go installed. Then, install the dependencies:

   ```bash
   go mod download
   ```

5. **Build Frontend Assets:**

   Install Tailwind CSS and build the CSS:

   ```bash
   cd web/static/css
   npx tailwindcss build tailwind.css -o output.css
   cd ../../../
   ```

6. **Start the Server:**

   ```bash
   go run cmd/server/main.go
   ```

   The application should be available at `http://localhost:8080`.

## Key Components

### 1. Main Server Entry Point

```typescript:cmd/server/main.go
package main

import (
    "log"
    "net/http"
    "os"

    "project/internal/auth"
    "project/internal/db"
    "project/internal/handlers"

    "github.com/joho/godotenv"
)

func main() {
    // Load environment variables
    err := godotenv.Load()
    if err != nil {
        log.Fatal("Error loading .env file")
    }

    // Initialize database
    dbConn, err := db.InitDB()
    if err != nil {
        log.Fatal(err)
    }
    defer dbConn.Close()

    // Initialize OAuth configuration
    oauthConfig := auth.InitOAuth()

    // Set up HTTP routes
    mux := http.NewServeMux()
    mux.HandleFunc("/", handlers.HomeHandler)
    mux.HandleFunc("/login", auth.LoginHandler(oauthConfig))
    mux.HandleFunc("/auth/callback", auth.CallbackHandler(oauthConfig, dbConn))

    // Serve static files
    fs := http.FileServer(http.Dir("./web/static"))
    mux.Handle("/static/", http.StripPrefix("/static/", fs))

    // Start the server
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    log.Printf("Server starting at :%s", port)
    err = http.ListenAndServe(":"+port, mux)
    if err != nil {
        log.Fatal(err)
    }
}
```

**Explanation:**
- Loads environment variables.
- Initializes the PostgreSQL database connection.
- Sets up OAuth configuration.
- Defines HTTP routes for home, login, and OAuth callbacks.
- Serves static files from the `web/static` directory.
- Starts the HTTP server on the specified port.

---

### 2. Database Initialization

```typescript:internal/db/db.go
package db

import (
    "database/sql"
    "fmt"
    "os"

    _ "github.com/lib/pq"
)

func InitDB() (*sql.DB, error) {
    dbURL := os.Getenv("DATABASE_URL")
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to database: %w", err)
    }

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    return db, nil
}
```

**Explanation:**
- Connects to the PostgreSQL database using the `DATABASE_URL` from environment variables.
- Pings the database to ensure the connection is established.

---

### 3. User Model

```typescript:internal/models/user.go
package models

type User struct {
    ID        int    `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email"`
    OAuthID   string `json:"oauth_id"`
    CreatedAt string `json:"created_at"`
}
```

**Explanation:**
- Defines the `User` struct representing a user in the system.
- Includes fields for ID, Name, Email, OAuth ID, and the creation timestamp.

---

### 4. OAuth Authentication

```typescript:internal/auth/oauth.go
package auth

import (
    "database/sql"
    "encoding/json"
    "fmt"
    "net/http"
    "os"

    "project/internal/models"

    "github.com/gorilla/sessions"
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

var store = sessions.NewCookieStore([]byte(os.Getenv("SESSION_SECRET")))

func InitOAuth() *oauth2.Config {
    return &oauth2.Config{
        RedirectURL:  os.Getenv("OAUTH_REDIRECT_URL"),
        ClientID:     os.Getenv("OAUTH_CLIENT_ID"),
        ClientSecret: os.Getenv("OAUTH_CLIENT_SECRET"),
        Scopes:       []string{"https://www.googleapis.com/auth/userinfo.email", "https://www.googleapis.com/auth/userinfo.profile"},
        Endpoint:     google.Endpoint,
    }
}

func LoginHandler(config *oauth2.Config) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        url := config.AuthCodeURL("state", oauth2.AccessTypeOffline)
        http.Redirect(w, r, url, http.StatusTemporaryRedirect)
    }
}

func CallbackHandler(config *oauth2.Config, db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        session, _ := store.Get(r, "auth-session")
        code := r.URL.Query().Get("code")
        token, err := config.Exchange(r.Context(), code)
        if err != nil {
            http.Error(w, "Failed to exchange token", http.StatusInternalServerError)
            return
        }

        // Retrieve user info from OAuth provider
        resp, err := http.Get("https://www.googleapis.com/oauth2/v2/userinfo?access_token=" + token.AccessToken)
        if err != nil {
            http.Error(w, "Failed to get user info", http.StatusInternalServerError)
            return
        }
        defer resp.Body.Close()

        var userInfo struct {
            ID    string `json:"id"`
            Name  string `json:"name"`
            Email string `json:"email"`
        }

        if err := json.NewDecoder(resp.Body).Decode(&userInfo); err != nil {
            http.Error(w, "Failed to decode user info", http.StatusInternalServerError)
            return
        }

        // Insert or update user in the database
        var userID int
        err = db.QueryRow(`
            INSERT INTO users (name, email, oauth_id)
            VALUES ($1, $2, $3)
            ON CONFLICT (oauth_id) DO UPDATE 
            SET name = EXCLUDED.name, email = EXCLUDED.email
            RETURNING id
        `, userInfo.Name, userInfo.Email, userInfo.ID).Scan(&userID)

        if err != nil {
            http.Error(w, "Failed to save user", http.StatusInternalServerError)
            return
        }

        // Set session
        session.Values["authenticated"] = true
        session.Values["user_id"] = userID
        session.Save(r, w)

        http.Redirect(w, r, "/", http.StatusSeeOther)
    }
}
```

**Explanation:**
- Initializes OAuth2 configuration using Google as the provider.
- `LoginHandler` redirects users to Google's OAuth consent page.
- `CallbackHandler` handles the OAuth callback, exchanges the code for a token, retrieves user info, saves or updates the user in the database, and sets session variables.

---

### 5. Home Handler

```typescript:internal/handlers/home.go
package handlers

import (
    "html/template"
    "net/http"
)

func HomeHandler(w http.ResponseWriter, r *http.Request) {
    tmpl, err := template.ParseFiles("web/templates/base.html", "web/templates/home.html")
    if err != nil {
        http.Error(w, "Unable to parse template", http.StatusInternalServerError)
        return
    }

    data := struct{
        Title string
        User  interface{}
    }{
        Title: "Home",
        User:  nil, // Replace with actual user data if authenticated
    }

    tmpl.ExecuteTemplate(w, "base", data)
}
```

**Explanation:**
- Parses the base and home HTML templates.
- Prepares data to be injected into the templates (e.g., page title, user information).
- Renders the `base` template with the provided data.

---

### 6. Database Migration

```sql:migrations/001_create_users_table.sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE NOT NULL,
    oauth_id VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Explanation:**
- Creates a `users` table with fields for ID, Name, Email, OAuth ID, and creation timestamp.
- Ensures that the `email` and `oauth_id` are unique.

---

### 7. Base HTML Template

```html:web/templates/base.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ .Title }}</title>
    <link href="/static/css/output.css" rel="stylesheet">
    <script src="/static/js/main.js" defer></script>
</head>
<body class="bg-gray-100">
    {{ template "navbar" . }}
    <div class="container mx-auto mt-4">
        {{ block "content" . }}{{ end }}
    </div>
</body>
</html>
```

**Explanation:**
- Defines the basic HTML structure.
- Includes Tailwind CSS and a main JavaScript file.
- Incorporates the `navbar` component.
- Provides a `content` block for individual pages to inject their content.

---

### 8. Navbar Component with Shadcn

```html:web/components/navbar.html
{{ define "navbar" }}
<nav class="bg-white shadow">
    <div class="container mx-auto px-4">
        <div class="flex justify-between">
            <div class="flex space-x-4">
                <a href="/" class="text-gray-700 px-3 py-2">Home</a>
                {{ if .User }}
                <a href="/profile" class="text-gray-700 px-3 py-2">Profile</a>
                {{ else }}
                <a href="/login" class="text-gray-700 px-3 py-2">Login</a>
                {{ end }}
            </div>
        </div>
    </div>
</nav>
{{ end }}
```

**Explanation:**
- Defines a reusable navbar component.
- Shows different links based on whether the user is authenticated.

---

### 9. Home Page Template

```html:web/templates/home.html
{{ define "content" }}
<h1 class="text-2xl font-bold">Welcome to the Project Template</h1>
<p class="mt-4">This is a starting point for your application.</p>

<!-- Example HTMX usage -->
<button hx-get="/some-endpoint" hx-target="#result" class="mt-4 bg-blue-500 text-white px-4 py-2 rounded">
    Load Content
</button>
<div id="result"></div>
{{ end }}
```

**Explanation:**
- Provides the content for the home page.
- Includes an example of HTMX usage to dynamically load content without a full page refresh.

---

### 10. Tailwind CSS Configuration

```javascript:web/static/css/tailwind.config.js
module.exports = {
  content: [
    './web/**/*.html',
    './web/components/**/*.html',
    // Add any additional paths as needed
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

**Explanation:**
- Configures Tailwind CSS to scan HTML files for class usage.
- Extend the theme or add plugins as necessary.

---

### 11. Main JavaScript

```javascript:web/static/js/main.js
// Example JavaScript file
console.log("Frontend assets loaded.");
```

**Explanation:**
- A placeholder for your main JavaScript logic.
- Can be expanded to include custom scripts or integrate Shadcn components.

---

## Deployment

To deploy the application, you can use Docker for containerization. Below is a simple `Dockerfile` and `docker-compose.yml` for setting up both the Go server and PostgreSQL database.

### Dockerfile

```dockerfile:Dockerfile
# Stage 1: Build the Go application
FROM golang:1.20-alpine AS builder

WORKDIR /app

# Install dependencies
COPY go.mod go.sum ./
RUN go mod download

# Copy the source code
COPY . .

# Build the application
RUN go build -o server cmd/server/main.go

# Stage 2: Run the application
FROM alpine:latest

WORKDIR /root/

# Install necessary packages
RUN apk --no-cache add ca-certificates

# Copy the binary and static files from the builder
COPY --from=builder /app/server .
COPY --from=builder /app/web /web

# Expose port
EXPOSE 8080

# Set environment variables (can be overridden)
ENV PORT=8080

# Command to run the server
CMD ["./server"]
```

**Explanation:**
- **Stage 1:** Builds the Go application using the official Golang image.
- **Stage 2:** Creates a minimal Alpine-based image, copies the built binary and static files, and sets the entry point.

---

### Docker Compose for PostgreSQL and Server

```yaml:docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14-alpine
    restart: always
    environment:
      POSTGRES_USER: youruser
      POSTGRES_PASSWORD: yourpassword
      POSTGRES_DB: yourdb
    volumes:
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  server:
    build: .
    depends_on:
      - db
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://youruser:yourpassword@db:5432/yourdb?sslmode=disable
      OAUTH_CLIENT_ID: your-client-id
      OAUTH_CLIENT_SECRET: your-client-secret
      OAUTH_REDIRECT_URL: http://localhost:8080/auth/callback
      SESSION_SECRET: your-session-secret

volumes:
  db-data:
```

**Explanation:**
- **db Service:**
  - Uses the official PostgreSQL Alpine image.
  - Sets up environment variables for the database user, password, and name.
  - Persists data using a Docker volume.
  - Exposes port `5432` for database connections.
  
- **server Service:**
  - Builds the Go server using the provided `Dockerfile`.
  - Depends on the `db` service to ensure the database is up before starting.
  - Maps port `8080` to the host.
  - Passes necessary environment variables, including database connection string and OAuth credentials.

---

### Building and Running with Docker Compose

1. **Ensure Docker and Docker Compose are installed.**

2. **Build and Start Services:**

   ```bash
   docker-compose up --build -d
   ```

3. **Apply Database Migrations:**

   You can run the migration inside the `db` container:

   ```bash
   docker exec -it project-template_db_1 psql -U youruser -d yourdb -f /migrations/001_create_users_table.sql
   ```

   *(Replace `project-template_db_1` with your actual db container name if different.)*

4. **Access the Application:**

   Open your browser and navigate to `http://localhost:8080`.

---

## Conclusion

This template offers a comprehensive starting point for building scalable and secure web applications with:

- **Go** for a high-performance backend.
- **PostgreSQL** as a reliable relational database.
- **HTMX** for dynamic frontend interactions without heavy JavaScript.
- **Tailwind CSS** for utility-first styling.
- **Shadcn** for pre-built UI components.
- **OAuth** for secure authentication.

Customize and expand upon this template to fit the specific needs of your project. Happy coding!