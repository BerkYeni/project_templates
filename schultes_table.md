# Schulte's Table App Development Guide

This document provides step-by-step instructions for building and deploying a Schulte's Table App. The backend is built using Go, serving the frontend built with React.js, Tailwind CSS, and Shadcn. The application logic leverages the `useReducer` React hook, and the app is hosted on Cloudflare VPS.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Backend Setup (Go)](#backend-setup-go)
4. [Frontend Setup (React.js, Tailwind, Shadcn)](#frontend-setup-reactjs-tailwind-shadcn)
5. [Application Logic with useReducer](#application-logic-with-usereducer)
6. [Deployment on Cloudflare VPS](#deployment-on-cloudflare-vps)
7. [Conclusion](#conclusion)

---

## Prerequisites

Before starting, ensure you have the following installed on your development machine:

- [Go](https://golang.org/dl/) (version 1.18 or later)
- [Node.js](https://nodejs.org/en/download/) (version 14 or later)
- [npm](https://www.npmjs.com/get-npm) or [Yarn](https://yarnpkg.com/getting-started/install)
- [Git](https://git-scm.com/downloads)
- A [Cloudflare account](https://dash.cloudflare.com/sign-up)

---

## Project Structure

Organize your project directory as follows:

```
schulte-table-app/
├── backend/
│   └── main.go
├── frontend/
│   ├── public/
│   └── src/
│       ├── components/
│       │   └── SchulteTable.tsx
│       ├── hooks/
│       │   └── useSchulteLogic.ts
│       ├── App.tsx
│       ├── index.tsx
│       └── ...
├── .gitignore
├── README.md
└── ...
```

---

## Backend Setup (Go)

The backend serves the React frontend as static files.

### 1. Initialize the Go Project

Navigate to the `backend` directory and initialize a new Go module.

```bash
cd schulte-table-app/backend
go mod init github.com/yourusername/schulte-table-app/backend
```

### 2. Create the Server

Create a `main.go` file to set up the HTTP server.

```go:backend/main.go
package main

import (
    "log"
    "net/http"
    "os"
)

func main() {
    fs := http.FileServer(http.Dir("../frontend/build"))
    http.Handle("/", fs)

    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    log.Printf("Server is running on port %s", port)
    err := http.ListenAndServe(":"+port, nil)
    if err != nil {
        log.Fatal(err)
    }
}
```

### 3. Build the Frontend for Production

Before running the backend server, build the frontend.

```bash
cd ../frontend
npm install
npm run build
```

### 4. Run the Backend Server

Navigate back to the `backend` directory and run the server.

```bash
cd ../backend
go run main.go
```

The server will serve the frontend at `http://localhost:8080`.

---

## Frontend Setup (React.js, Tailwind, Shadcn)

### 1. Initialize the React Project

Navigate to the `frontend` directory and create a new React app.

```bash
cd schulte-table-app/frontend
npx create-react-app . --template typescript
```

### 2. Install Dependencies

Install Tailwind CSS and Shadcn components.

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

npm install @shadcn/ui
```

### 3. Configure Tailwind CSS

Update `tailwind.config.js`:

```javascript
// frontend/tailwind.config.js
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

Import Tailwind in `src/index.css`:

```css
/* frontend/src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 4. Create SchulteTable Component

Create a `SchulteTable.tsx` component to display the table.

```typescript:frontend/src/components/SchulteTable.tsx
import React from 'react';

const SchulteTable: React.FC = () => {
    // Component implementation
    return (
        <div>
            {/* Schulte's Table UI */}
        </div>
    );
}

export default SchulteTable;
```

---

## Application Logic with useReducer

Implement the application logic using the `useReducer` hook for state management.

### 1. Define State and Actions

Create a custom hook to manage the Schulte table state.

```typescript:frontend/src/hooks/useSchulteLogic.ts
import { useReducer } from 'react';

type State = {
    numbers: number[];
    selectedNumber: number | null;
    // Additional state properties
};

type Action =
    | { type: 'SELECT_NUMBER'; payload: number }
    | { type: 'RESET_TABLE' }
    // Additional actions

const initialState: State = {
    numbers: generateRandomNumbers(),
    selectedNumber: null,
    // Initialize other state properties
};

function reducer(state: State, action: Action): State {
    switch (action.type) {
        case 'SELECT_NUMBER':
            return { ...state, selectedNumber: action.payload };
        case 'RESET_TABLE':
            return { ...initialState, numbers: generateRandomNumbers() };
        default:
            return state;
    }
}

function useSchulteLogic() {
    const [state, dispatch] = useReducer(reducer, initialState);
    return { state, dispatch };
}

function generateRandomNumbers(): number[] {
    const numbers = Array.from({ length: 25 }, (_, i) => i + 1);
    return shuffle(numbers);
}

function shuffle(array: number[]): number[] {
    return array.sort(() => Math.random() - 0.5);
}

export default useSchulteLogic;
```

### 2. Integrate Logic into SchulteTable Component

```typescript:frontend/src/components/SchulteTable.tsx
import React from 'react';
import useSchulteLogic from '../hooks/useSchulteLogic';

const SchulteTable: React.FC = () => {
    const { state, dispatch } = useSchulteLogic();

    const handleNumberClick = (number: number) => {
        dispatch({ type: 'SELECT_NUMBER', payload: number });
        // Additional logic
    };

    return (
        <div className="grid grid-cols-5 gap-4">
            {state.numbers.map((number) => (
                <button
                    key={number}
                    onClick={() => handleNumberClick(number)}
                    className="p-4 bg-blue-500 text-white rounded"
                >
                    {number}
                </button>
            ))}
        </div>
    );
}

export default SchulteTable;
```

### 3. Update App Component

Integrate the `SchulteTable` component into the main application.

```typescript:frontend/src/App.tsx
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

---

## Deployment on Cloudflare VPS

### 1. Build the Frontend for Production

Ensure the frontend is built and ready to be served by the Go backend.

```bash
cd schulte-table-app/frontend
npm run build
```

### 2. Set Up Cloudflare VPS

- Log in to your Cloudflare account.
- Navigate to the **Cloudflare Pages** or **Cloudflare Workers** section, depending on your preference.
- For VPS, you might use **Cloudflare's Spectrum** or set up a traditional VPS instance with appropriate settings.

### 3. Configure DNS

- Point your domain's DNS records to the Cloudflare VPS IP address.
- Ensure that the necessary ports (e.g., 80 for HTTP, 443 for HTTPS) are open and properly routed.

### 4. Deploy the Go Backend

- SSH into your Cloudflare VPS.
- Install Go if not already installed.

```bash
sudo apt update
sudo apt install golang
```

- Clone your project repository.

```bash
git clone https://github.com/yourusername/schulte-table-app.git
cd schulte-table-app/backend
```

- Build and run the Go server.

```bash
go build -o server
./server
```

- Use a process manager like `systemd` or `pm2` to keep the server running.

### 5. Configure HTTPS

- Use Cloudflare's SSL/TLS features to secure your application.
- In the Cloudflare dashboard, navigate to the **SSL/TLS** section and configure as needed.

---

## Conclusion

By following this guide, developers can efficiently build and deploy a Schulte's Table App using Go for the backend and React.js with Tailwind CSS and Shadcn for the frontend. The application leverages the `useReducer` hook for state management and is hosted on Cloudflare VPS for reliable performance. Customize and expand upon this foundation to add more features and enhance the user experience.

---

# Additional Resources

- [Go Documentation](https://golang.org/doc/)
- [React.js Documentation](https://reactjs.org/docs/getting-started.html)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [Shadcn Components](https://shadcn.com/)
- [Cloudflare VPS Guides](https://developers.cloudflare.com/)

# License

This project is licensed under the MIT License.

# Feedback

For any issues or feature requests, please open an issue on the [GitHub repository](https://github.com/yourusername/schulte-table-app).

# Acknowledgments

- Inspired by the classic Schulte tables used for improving attention and speed.

# Contact

For further inquiries, contact [your.email@example.com](mailto:your.email@example.com).

# Happy Coding!

---

**Note:** Replace placeholders like `yourusername`, `your.email@example.com`, and repository links with your actual information.