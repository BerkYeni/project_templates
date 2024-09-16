# Flask Backend Project Template

This project template provides a solid foundation for developers to quickly build and deploy web applications using Flask for the backend, SQLite for the database, and a modern frontend stack including HTMX, Tailwind CSS, and Shadcn. It also incorporates OAuth for secure authentication.

## Project Structure

```
flask_project_template/
├── app/
│   ├── __init__.py
│   ├── auth.py
│   ├── models.py
│   ├── routes.py
│   ├── static/
│   │   ├── css/
│   │   │   └── tailwind.css
│   │   └── js/
│   │       └── main.js
│   └── templates/
│       ├── base.html
│       ├── index.html
│       └── login.html
├── migrations/
├── instance/
│   └── config.py
├── requirements.txt
├── tailwind.config.js
├── package.json
└── run.py
```

## Setup Instructions

1. **Clone the Repository**

   ```bash
   git clone https://github.com/yourusername/flask_project_template.git
   cd flask_project_template
   ```

2. **Create a Virtual Environment and Install Dependencies**

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

3. **Set Up Environment Variables**

   Create a `.env` file in the `instance/` directory and add your OAuth credentials.

4. **Initialize the Database**

   ```bash
   flask db init
   flask db migrate -m "Initial migration."
   flask db upgrade
   ```

5. **Build Frontend Assets**

   ```bash
   npm install
   npx tailwindcss -i ./app/static/css/tailwind.css -o ./app/static/css/output.css --watch
   ```

6. **Run the Application**

   ```bash
   flask run
   ```

## Key Components

### 1. Application Factory

```python:app/__init__.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from .auth import oauth_blueprint

db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()

def create_app():
    app = Flask(__name__, instance_relative_config=True)
    app.config.from_object('config.Config')
    app.register_blueprint(oauth_blueprint)
    
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)
    
    with app.app_context():
        from . import routes
        db.create_all()
    
    return app
```

### 2. Configuration

```python:instance/config.py
import os

class Config:
    SECRET_KEY = os.getenv('SECRET_KEY', 'your-secret-key')
    SQLALCHEMY_DATABASE_URI = 'sqlite:///site.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    OAUTH_CREDENTIALS = {
        'github': {
            'client_id': os.getenv('GITHUB_CLIENT_ID'),
            'client_secret': os.getenv('GITHUB_CLIENT_SECRET')
        }
    }
```

### 3. Models

```python:app/models.py
from . import db, login_manager
from flask_login import UserMixin

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(150), nullable=False, unique=True)
    oauth_provider = db.Column(db.String(50))
    oauth_id = db.Column(db.String(100), unique=True)
    
    def __repr__(self):
        return f'<User {self.username}>'
```

### 4. Authentication (OAuth)

```python:app/auth.py
from flask import Blueprint, redirect, url_for
from flask_dance.contrib.github import make_github_blueprint, github
from . import db
from .models import User
from flask_login import login_user

oauth_blueprint = Blueprint('auth', __name__)

github_bp = make_github_blueprint(
    client_id='YOUR_GITHUB_CLIENT_ID',
    client_secret='YOUR_GITHUB_CLIENT_SECRET',
)
oauth_blueprint.register_blueprint(github_bp, url_prefix='/login')

@oauth_blueprint.route('/login/github')
def login_github():
    if not github.authorized:
        return redirect(url_for('github.login'))
    resp = github.get('/user')
    if resp.ok:
        github_info = resp.json()
        username = github_info['login']
        oauth_id = str(github_info['id'])
        user = User.query.filter_by(oauth_id=oauth_id).first()
        if not user:
            user = User(username=username, oauth_provider='github', oauth_id=oauth_id)
            db.session.add(user)
            db.session.commit()
        login_user(user)
        return redirect(url_for('routes.index'))
    return redirect(url_for('routes.index'))
```

### 5. Routes

```python:app/routes.py
from flask import render_template, redirect, url_for
from flask_login import current_user, logout_user
from . import db
from .models import User
from .auth import oauth_blueprint

from flask import Blueprint

main = Blueprint('routes', __name__)

@main.route('/')
def index():
    return render_template('index.html')

@main.route('/logout')
def logout():
    logout_user()
    return redirect(url_for('routes.index'))
```

### 6. Templates

#### Base Template

```html:app/templates/base.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ title if title else "Flask App" }}</title>
    <link href="{{ url_for('static', filename='css/output.css') }}" rel="stylesheet">
    <script src="https://unpkg.com/htmx.org@1.9.2"></script>
</head>
<body class="bg-gray-100">
    <nav class="bg-white shadow">
        <div class="max-w-7xl mx-auto px-4">
            <div class="flex justify-between">
                <div class="flex space-x-4">
                    <a href="{{ url_for('routes.index') }}" class="py-5 px-3 text-gray-700">Home</a>
                </div>
                <div class="flex space-x-4">
                    {% if current_user.is_authenticated %}
                        <span class="py-5 px-3">{{ current_user.username }}</span>
                        <a href="{{ url_for('routes.logout') }}" class="py-5 px-3 text-gray-700">Logout</a>
                    {% else %}
                        <a href="{{ url_for('auth.login_github') }}" class="py-5 px-3 text-gray-700">Login with GitHub</a>
                    {% endif %}
                </div>
            </div>
        </div>
    </nav>
    
    <div class="container mx-auto mt-4">
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

#### Index Page

```html:app/templates/index.html
{% extends "base.html" %}

{% block content %}
<h1 class="text-2xl font-bold">Welcome to the Flask Project Template!</h1>
{% if current_user.is_authenticated %}
    <p class="mt-4">Hello, {{ current_user.username }}!</p>
{% else %}
    <p class="mt-4">Please log in to continue.</p>
{% endif %}
{% endblock %}
```

#### Login Page (Optional)

```html:app/templates/login.html
{% extends "base.html" %}

{% block content %}
<h2 class="text-xl">Login</h2>
<a href="{{ url_for('auth.login_github') }}" class="mt-4 px-4 py-2 bg-blue-500 text-white rounded">Login with GitHub</a>
{% endblock %}
```

### 7. Static Files

#### Tailwind CSS

```css:app/static/css/tailwind.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

#### Main JavaScript (HTMX Integration)

```javascript:app/static/js/main.js
// Your custom JavaScript if needed
```

### 8. Run Script

```python:run.py
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
```

### 9. Dependencies

```plaintext:requirements.txt
Flask
Flask-SQLAlchemy
Flask-Migrate
Flask-Login
Flask-Dance
python-dotenv
```

### 10. Tailwind Configuration

```javascript:tailwind.config.js
module.exports = {
  content: ['./app/templates/**/*.html', './app/static/js/**/*.js'],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### 11. Package Configuration

```json:package.json
{
  "name": "flask_project_template",
  "version": "1.0.0",
  "scripts": {
    "build:css": "tailwindcss -i ./app/static/css/tailwind.css -o ./app/static/css/output.css --watch"
  },
  "devDependencies": {
    "tailwindcss": "^3.0.0"
  }
}
```

## Features

- **Flask Backend:** A lightweight and scalable backend framework.
- **SQLite Database:** Easy-to-use relational database for development and small-scale applications.
- **HTMX:** Enhances frontend interactivity with minimal JavaScript.
- **Tailwind CSS:** Utility-first CSS framework for rapid UI development.
- **Shadcn:** (Note: Shadcn is primarily a React-based component library. For integration with Flask, consider using similar styled components or adapt its design principles.)
- **OAuth Authentication:** Secure user authentication using GitHub OAuth.

## Notes

- **Shadcn Integration:** Since Shadcn is designed for React, integrating it directly with Flask requires setting up a React frontend. Alternatively, you can mimic its styling using Tailwind CSS within your Flask templates.
- **Environment Variables:** Ensure that sensitive information like `SECRET_KEY`, `GITHUB_CLIENT_ID`, and `GITHUB_CLIENT_SECRET` are stored securely, preferably using environment variables.
- **Deployment:** For production deployment, consider using a more robust database system and configuring a WSGI server like Gunicorn.

## Conclusion

This template serves as a comprehensive starting point for building Flask-based web applications with modern frontend technologies and secure authentication mechanisms. Customize and extend it based on your project requirements to accelerate your development process.