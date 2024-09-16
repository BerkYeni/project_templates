# Flask Project Template

This template provides a streamlined backend setup using Flask, SQLite for the database, HTMX, Tailwind CSS, and Shadcn for the frontend, OAuth for authentication, and is configured for deployment on a Cloudflare VPS. Developers can quickly get started with this structure to build and deploy their applications efficiently.

## Project Structure

```
project/
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── routes.py
│   ├── auth.py
│   ├── templates/
│   │   ├── base.html
│   │   ├── index.html
│   │   └── login.html
│   └── static/
│       ├── css/
│       │   └── styles.css
│       ├── js/
│       │   └── main.js
│       └── img/
├── migrations/
├── config.py
├── requirements.txt
├── run.py
├── tailwind.config.js
├── postcss.config.js
├── package.json
├── package-lock.json
└── README.md
```

## Detailed File Structure and Code

### `app/__init__.py`

Initializes the Flask application, sets up extensions, and registers blueprints.

```python:app/__init__.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from config import Config

db = SQLAlchemy()
migrate = Migrate()
login = LoginManager()
login.login_view = 'auth.login'

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    migrate.init_app(app, db)
    login.init_app(app)

    from app.routes import main_bp
    from app.auth import auth_bp
    app.register_blueprint(main_bp)
    app.register_blueprint(auth_bp, url_prefix='/auth')

    return app
```

### `app/models.py`

Defines the database models using SQLAlchemy.

```python:app/models.py
from app import db, login
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), index=True, unique=True, nullable=False)
    email = db.Column(db.String(120), index=True, unique=True, nullable=False)
    password_hash = db.Column(db.String(128))

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)
 
    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

@login.user_loader
def load_user(id):
    return User.query.get(int(id))
```

### `app/routes.py`

Handles the main routes of the application.

```python:app/routes.py
from flask import Blueprint, render_template
from flask_login import login_required, current_user

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
@login_required
def index():
    return render_template('index.html', user=current_user)
```

### `app/auth.py`

Manages authentication routes and OAuth implementation.

```python:app/auth.py
from flask import Blueprint, render_template, redirect, url_for, flash
from flask_login import login_user, logout_user, login_required
from app.models import User
from app import db
from werkzeug.urls import url_parse

auth_bp = Blueprint('auth', __name__)

@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    # Implement OAuth login logic here
    pass

@auth_bp.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('auth.login'))
```

### `app/templates/base.html`

Base HTML template using Tailwind CSS and HTMX.

```html:app/templates/base.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ title if title else "Flask App" }}</title>
    <link href="{{ url_for('static', filename='css/styles.css') }}" rel="stylesheet">
    <script src="{{ url_for('static', filename='js/main.js') }}" defer></script>
    <script src="https://unpkg.com/htmx.org@1.9.2"></script>
</head>
<body class="bg-gray-100">
    <nav class="bg-white shadow mb-4">
        <div class="container mx-auto px-4">
            <div class="flex justify-between">
                <div>
                    <a href="{{ url_for('main.index') }}" class="text-xl font-bold">My Flask App</a>
                </div>
                <div>
                    {% if current_user.is_authenticated %}
                        <span class="mr-4">Hello, {{ current_user.username }}</span>
                        <a href="{{ url_for('auth.logout') }}" class="text-blue-500">Logout</a>
                    {% else %}
                        <a href="{{ url_for('auth.login') }}" class="text-blue-500">Login</a>
                    {% endif %}
                </div>
            </div>
        </div>
    </nav>
    <div class="container mx-auto px-4">
        {% with messages = get_flashed_messages() %}
            {% if messages %}
                <ul>
                {% for message in messages %}
                    <li>{{ message }}</li>
                {% endfor %}
                </ul>
            {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

### `app/templates/index.html`

Homepage template extending the base layout.

```html:app/templates/index.html
{% extends "base.html" %}

{% block content %}
<h1 class="text-2xl font-bold mb-4">Welcome, {{ user.username }}!</h1>
<p>This is your dashboard.</p>
{% endblock %}
```

### `app/templates/login.html`

Login page template.

```html:app/templates/login.html
{% extends "base.html" %}

{% block content %}
<h2 class="text-xl font-semibold mb-4">Login</h2>
<!-- OAuth login buttons can be added here -->
<a href="{{ url_for('auth.oauth_login') }}" class="btn btn-primary">Login with OAuth</a>
{% endblock %}
```

### `app/static/css/styles.css`

Tailwind CSS initial setup.

```css:app/static/css/styles.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom styles can be added here */
```

### `app/static/js/main.js`

Main JavaScript file for frontend interactivity.

```javascript:app/static/js/main.js
// Custom JavaScript can be added here
```

### `config.py`

Configuration settings for the Flask application.

```python:config.py
import os

basedir = os.path.abspath(os.path.dirname(__file__))

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'you-will-never-guess'
    SQLALCHEMY_DATABASE_URI = 'sqlite:///' + os.path.join(basedir, 'app.db')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    # OAuth configuration
    OAUTH_CLIENT_ID = os.environ.get('OAUTH_CLIENT_ID')
    OAUTH_CLIENT_SECRET = os.environ.get('OAUTH_CLIENT_SECRET')
```

### `run.py`

Entry point to run the Flask application.

```python:run.py
from app import create_app, db
from app.models import User

app = create_app()

@app.shell_context_processor
def make_shell_context():
    return {'db': db, 'User': User}

if __name__ == "__main__":
    app.run()
```

### `requirements.txt`

Python dependencies for the project.

```plaintext:requirements.txt
Flask
Flask-SQLAlchemy
Flask-Migrate
Flask-Login
oauthlib
requests
Flask-Dance
```

### `tailwind.config.js`

Tailwind CSS configuration.

```javascript:tailwind.config.js
module.exports = {
  content: [
    './app/templates/**/*.html',
    './app/static/js/**/*.js',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### `postcss.config.js`

PostCSS configuration for Tailwind.

```javascript:postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

### `package.json`

NPM dependencies and scripts for frontend tooling.

```json:package.json
{
  "name": "flask-app",
  "version": "1.0.0",
  "scripts": {
    "build:css": "postcss app/static/css/styles.css -o app/static/css/output.css"
  },
  "devDependencies": {
    "autoprefixer": "^10.4.12",
    "postcss": "^8.4.18",
    "tailwindcss": "^3.2.4"
  }
}
```

## Setup Instructions

### Prerequisites

- **Python 3.7+**
- **Node.js and npm**
- **Cloudflare VPS account**

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/your-repo.git
cd your-repo
```

### 2. Set Up the Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Python Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Environment Variables

Create a `.env` file in the root directory and add the following:

```env
SECRET_KEY=your_secret_key
OAUTH_CLIENT_ID=your_oauth_client_id
OAUTH_CLIENT_SECRET=your_oauth_client_secret
```

Load the environment variables in your shell:

```bash
export SECRET_KEY=your_secret_key
export OAUTH_CLIENT_ID=your_oauth_client_id
export OAUTH_CLIENT_SECRET=your_oauth_client_secret
```

### 5. Initialize the Database

```bash
flask db init
flask db migrate -m "Initial migration."
flask db upgrade
```

### 6. Set Up Frontend Dependencies

Install Node.js dependencies:

```bash
npm install
```

Build Tailwind CSS:

```bash
npm run build:css
```

### 7. Run the Application Locally

```bash
flask run
```

Visit `http://localhost:5000` in your browser.

## Deployment on Cloudflare VPS

### 1. Provision a VPS

- Sign in to your Cloudflare account.
- Provision a new VPS instance.
- Note the IP address and SSH credentials.

### 2. Secure the Server

- **Update Packages:**

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

- **Install Required Software:**

  ```bash
  sudo apt install python3-pip python3-venv nginx git
  ```

### 3. Clone the Repository on the Server

```bash
git clone https://github.com/yourusername/your-repo.git
cd your-repo
```

### 4. Set Up the Virtual Environment and Install Dependencies

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 5. Set Environment Variables

Export necessary environment variables:

```bash
export SECRET_KEY=your_secret_key
export OAUTH_CLIENT_ID=your_oauth_client_id
export OAUTH_CLIENT_SECRET=your_oauth_client_secret
```

### 6. Initialize the Database

```bash
flask db upgrade
```

### 7. Set Up Frontend Assets

```bash
npm install
npm run build:css
```

### 8. Configure Gunicorn

Install Gunicorn:

```bash
pip install gunicorn
```

Create a Gunicorn systemd service file `/etc/systemd/system/gunicorn.service`:

```ini
[Unit]
Description=Gunicorn instance to serve Flask App
After=network.target

[Service]
User=yourusername
Group=www-data
WorkingDirectory=/home/yourusername/your-repo
Environment="PATH=/home/yourusername/your-repo/venv/bin"
ExecStart=/home/yourusername/your-repo/venv/bin/gunicorn -w 3 run:app -b unix:app.sock

[Install]
WantedBy=multi-user.target
```

Start and enable Gunicorn:

```bash
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

### 9. Configure Nginx

Create an Nginx server block `/etc/nginx/sites-available/your-repo`:

```nginx
server {
    listen 80;
    server_name your_domain.com;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/yourusername/your-repo/app.sock;
    }

    location /static/ {
        alias /home/yourusername/your-repo/app/static/;
    }
}
```

Enable the server block and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/your-repo /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
```

### 10. Secure the Application with SSL

Use Cloudflare's SSL services to secure your domain.

## Additional Notes

- **OAuth Implementation:** Customize the `/auth/login` route in `app/auth.py` to integrate with your chosen OAuth provider using libraries like `Flask-Dance` or `Authlib`.
  
- **Frontend Enhancements:** Utilize HTMX for dynamic content loading without full page reloads, Tailwind CSS for utility-first styling, and Shadcn for pre-built UI components.

- **Environment Management:** Consider using tools like `python-dotenv` to manage environment variables more effectively.

- **Scaling:** For larger applications, consider using a more robust database like PostgreSQL and implementing caching mechanisms.

---

This template provides a solid foundation for building and deploying Flask applications with modern frontend technologies and secure authentication mechanisms. Customize and extend it based on your project's specific requirements.