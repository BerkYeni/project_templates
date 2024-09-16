# Django Project Template with PostgreSQL, HTMX, Tailwind, ShadCN, OAuth, and Cloudflare VPS Deployment

This template provides a comprehensive setup for building Django-based backend projects with PostgreSQL, HTMX, Tailwind CSS, ShadCN for the frontend, OAuth for authentication, and deployment on Cloudflare VPS. Follow the instructions and utilize the provided code snippets to quickly scaffold your project.

## Project Structure

```
myproject/
├── backend/
│   ├── manage.py
│   ├── myproject/
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   ├── wsgi.py
│   │   └── asgi.py
│   ├── app/
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── templates/
│   │   │   └── app/
│   │   │       └── index.html
│   │   └── static/
│   │       ├── css/
│   │       ├── js/
│   │       └── images/
├── frontend/
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   └── src/
│       ├── index.html
│       ├── styles.css
│       └── components/
│           └── ExampleComponent.tsx
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

## Setup Instructions

### 1. Initialize the Django Project

First, set up a virtual environment and install Django.

```bash
python3 -m venv venv
source venv/bin/activate
pip install django psycopg2-binary django-allauth
```

Create the Django project and app:

```bash
django-admin startproject myproject backend
cd backend
python manage.py startapp app
```

### 2. Configure PostgreSQL

Ensure you have PostgreSQL installed and create a database.

```sql
CREATE DATABASE myproject_db;
CREATE USER myproject_user WITH PASSWORD 'securepassword';
ALTER ROLE myproject_user SET client_encoding TO 'utf8';
ALTER ROLE myproject_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE myproject_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE myproject_db TO myproject_user;
```

Update `backend/myproject/settings.py`:

```python:backend/myproject/settings.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

# ... other settings ...

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myproject_db',
        'USER': 'myproject_user',
        'PASSWORD': 'securepassword',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

# Authentication
AUTHENTICATION_BACKENDS = (
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
)

INSTALLED_APPS = [
    # ... other apps ...
    'django.contrib.sites',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    # Providers
    'allauth.socialaccount.providers.google',
    # Your app
    'app',
]

SITE_ID = 1

# HTMX settings
INSTALLED_APPS += ['django_htmx']
MIDDLEWARE = [
    'django_htmx.middleware.HtmxMiddleware',
    # ... other middleware ...
]
```

### 3. Set Up OAuth Authentication

Using `django-allauth` for OAuth.

```bash
pip install django-allauth
```

Update `backend/myproject/urls.py`:

```python:backend/myproject/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('allauth.urls')),
    path('', include('app.urls')),
]
```

### 4. Create Views and Templates with HTMX

```python:backend/app/views.py
from django.shortcuts import render
from django.contrib.auth.decorators import login_required

@login_required
def index(request):
    return render(request, 'app/index.html')
```

```python:backend/app/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

```html:backend/app/templates/app/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>MyProject</title>
    <link href="{% static 'css/styles.css' %}" rel="stylesheet">
    <script src="https://unpkg.com/htmx.org@1.9.2"></script>
</head>
<body class="bg-gray-100">
    <div class="container mx-auto p-4">
        <h1 class="text-2xl font-bold">Welcome to MyProject</h1>
        <div hx-get="/some-endpoint" hx-trigger="click" class="mt-4">
            Click me to load content via HTMX
        </div>
    </div>
</body>
</html>
```

### 5. Integrate Tailwind CSS and ShadCN

Initialize Tailwind CSS in the `frontend` directory.

```bash
cd ../frontend
npm init -y
npm install tailwindcss postcss autoprefixer
npx tailwindcss init
```

Configure `frontend/tailwind.config.js`:

```javascript:frontend/tailwind.config.js
module.exports = {
  content: [
    './src/**/*.{html,js,tsx}',
    '../../backend/app/templates/**/*.html',
  ],
  theme: {
    extend: {},
  },
  plugins: [
    require('@shadcn/ui'),
  ],
}
```

Create `frontend/postcss.config.js`:

```javascript:frontend/postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

Create `frontend/src/styles.css`:

```css:frontend/src/styles.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom styles */
```

Build Tailwind CSS:

```bash
npx tailwindcss -o ../../backend/app/static/css/styles.css --watch
```

### 6. Add ShadCN Components

Install ShadCN components as per their documentation. Here's an example of a React component if you're using ShadCN with React.

```typescript:frontend/src/components/ExampleComponent.tsx
import React from 'react';
import { Button } from '@shadcn/ui';

export function ExampleComponent() {
    return (
        <div>
            <Button variant="primary">Click Me</Button>
        </div>
    );
}
```

### 7. Dockerize the Project

Create a `Dockerfile` in the root directory:

```dockerfile:Dockerfile
# Use official Python image as base
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory
WORKDIR /code

# Install dependencies
COPY requirements.txt /code/
RUN pip install --upgrade pip
RUN pip install -r requirements.txt

# Copy project
COPY backend /code/backend
COPY frontend /code/frontend

# Collect static files
RUN python backend/manage.py collectstatic --noinput

# Run the application
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

Create `docker-compose.yml`:

```yaml:docker-compose.yml
version: '3.9'

services:
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_DB=myproject_db
      - POSTGRES_USER=myproject_user
      - POSTGRES_PASSWORD=securepassword

  web:
    build: .
    command: gunicorn myproject.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - ./backend:/code/backend
      - ./frontend:/code/frontend
    ports:
      - "8000:8000"
    depends_on:
      - db

volumes:
  postgres_data:
```

### 8. Deployment on Cloudflare VPS

1. **Provision a VPS**: Choose a Cloudflare-compatible VPS provider.

2. **Configure DNS**: Point your domain to the VPS IP using Cloudflare DNS settings.

3. **Install Docker and Docker Compose** on the VPS:

    ```bash
    sudo apt update
    sudo apt install docker.io docker-compose -y
    sudo systemctl enable docker
    ```

4. **Transfer Project Files**: Use `git` or `scp` to transfer your project to the VPS.

5. **Set Environment Variables**: Securely set environment variables for Django settings.

6. **Run Docker Compose**:

    ```bash
    docker-compose up -d
    ```

7. **Configure SSL**: Use Cloudflare's SSL features for secure HTTPS connections.

## Dependencies

Add the following to `requirements.txt`:

```plaintext:requirements.txt
Django>=4.2
psycopg2-binary
django-allauth
django-htmx
gunicorn
```

## Additional Configuration

### Settings for OAuth Providers

Configure OAuth providers like Google in `backend/myproject/settings.py`:

```python:backend/myproject/settings.py
# Add to INSTALLED_APPS
INSTALLED_APPS += [
    'allauth.socialaccount.providers.google',
]

# Configure social account providers
SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'SCOPE': [
            'profile',
            'email',
        ],
        'AUTH_PARAMS': {
            'access_type': 'online',
        }
    }
}

LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
```

### Static Files Collection

Ensure that static files are collected properly during deployment.

```bash
python backend/manage.py collectstatic
```

### Security Settings

Update `settings.py` for security in production:

```python:backend/myproject/settings.py
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com']

# Security settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## Running the Project Locally

1. **Start Docker Containers**:

    ```bash
    docker-compose up --build
    ```

2. **Apply Migrations**:

    ```bash
    docker-compose exec web python backend/manage.py migrate
    ```

3. **Create a Superuser**:

    ```bash
    docker-compose exec web python backend/manage.py createsuperuser
    ```

4. **Access the Application**: Navigate to `http://localhost:8000` in your browser.

## Conclusion

This template sets up a robust foundation for Django projects with modern frontend tools and OAuth authentication. Customize the components as per your project requirements and leverage Docker for seamless deployment on Cloudflare VPS.