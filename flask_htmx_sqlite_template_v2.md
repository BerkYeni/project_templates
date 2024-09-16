# Flask Project Template with SQLite, OAuth, HTMX, Tailwind, and Shadcn

This project template provides a solid foundation for building a Flask backend with SQLite, OAuth authentication, and a modern frontend using HTMX, Tailwind CSS, and Shadcn components. Follow the structure and code examples below to quickly set up your project for deployment.

## Project Structure

```
my_project/
├── app/
│   ├── __init__.py
│   ├── auth/
│   │   ├── __init__.py
│   │   └── routes.py
│   ├── models.py
│   ├── static/
│   │   ├── css/
│   │   │   └── styles.css
│   │   └── js/
│   │       └── scripts.js
│   ├── templates/
│   │   ├── base.html
│   │   ├── index.html
│   │   └── login.html
│   └── utils.py
├── migrations/
├── requirements.txt
├── tailwind.config.js
├── shadcn.config.js
├── run.py
└── README.md
```

## Setup Instructions

1. **Clone the Repository**

   ```bash
   git clone https://github.com/yourusername/my_project.git
   cd my_project
   ```

2. **Create a Virtual Environment and Install Dependencies**

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

3. **Set Up Environment Variables**

   Create a `.env` file in the root directory and add your OAuth credentials and Flask configurations.

   ```env
   FLASK_APP=run.py
   FLASK_ENV=development
   SECRET_KEY=your_secret_key
   OAUTH_CLIENT_ID=your_oauth_client_id
   OAUTH_CLIENT_SECRET=your_oauth_client_secret
   ```

4. **Initialize the Database**

   ```bash
   flask db init
   flask db migrate -m "Initial migration."
   flask db upgrade
   ```

5. **Run the Development Server**

   ```bash
   flask run
   ```

## Dependencies

- **Backend:**
  - Flask
  - Flask-Login
  - Flask-OAuthlib
  - SQLAlchemy
  - Flask-Migrate

- **Frontend:**
  - HTMX
  - Tailwind CSS
  - Shadcn Components

## Detailed Code Structure

### `run.py`

This is the entry point of the Flask application.

```python:run.py
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
```

### `app/__init__.py`

Initializes the Flask app, database, and OAuth.

```python:app/__init__.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from .auth.routes import auth_bp
from .utils import configure_oauth

db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()

def create_app():
    app = Flask(__name__)
    app.config['SECRET_KEY'] = 'your_secret_key'
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///site.db'

    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)

    app.register_blueprint(auth_bp)

    configure_oauth(app)

    from . import models

    return app
```

### `app/models.py`

Defines the database models.

```python:app/models.py
from . import db
from flask_login import UserMixin

class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    oauth_provider = db.Column(db.String(50), nullable=False)
    oauth_id = db.Column(db.String(100), unique=True, nullable=False)
    username = db.Column(db.String(150), nullable=False)
    email = db.Column(db.String(150), unique=True, nullable=False)
```

### `app/auth/routes.py`

Handles authentication routes using OAuth.

```python:app/auth/routes.py
from flask import Blueprint, redirect, url_for
from flask_login import login_user, logout_user
from ..models import User
from .. import db, login_manager
from ..utils import oauth

auth_bp = Blueprint('auth', __name__)

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

@auth_bp.route('/login')
def login():
    return oauth.google.authorize_redirect(url_for('auth.authorize', _external=True))

@auth_bp.route('/authorize')
def authorize():
    token = oauth.google.authorize_access_token()
    user_info = oauth.google.parse_id_token(token)
    user = User.query.filter_by(oauth_id=user_info['sub']).first()
    if not user:
        user = User(
            oauth_provider='google',
            oauth_id=user_info['sub'],
            username=user_info['name'],
            email=user_info['email']
        )
        db.session.add(user)
        db.session.commit()
    login_user(user)
    return redirect(url_for('main.index'))

@auth_bp.route('/logout')
def logout():
    logout_user()
    return redirect(url_for('main.index'))
```

### `app/utils.py`

Configures OAuth using `Authlib`.

```python:app/utils.py
from authlib.integrations.flask_client import OAuth

oauth = OAuth()

def configure_oauth(app):
    oauth.init_app(app)
    oauth.register(
        name='google',
        client_id=app.config['OAUTH_CLIENT_ID'],
        client_secret=app.config['OAUTH_CLIENT_SECRET'],
        access_token_url='https://accounts.google.com/o/oauth2/token',
        authorize_url='https://accounts.google.com/o/oauth2/auth',
        client_kwargs={'scope': 'openid email profile'},
    )
```

### `app/templates/base.html`

Base HTML template using Tailwind and HTMX.

```html:app/templates/base.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ title if title else "My Project" }}</title>
    <link href="{{ url_for('static', filename='css/styles.css') }}" rel="stylesheet">
    <script src="https://unpkg.com/htmx.org@1.8.4"></script>
</head>
<body class="bg-gray-100">
    <nav class="bg-white shadow">
        <div class="container mx-auto px-4 py-4 flex justify-between">
            <a href="{{ url_for('main.index') }}" class="text-xl font-bold">My Project</a>
            <div>
                {% if current_user.is_authenticated %}
                    <span class="mr-4">Hello, {{ current_user.username }}!</span>
                    <a href="{{ url_for('auth.logout') }}" class="text-blue-500">Logout</a>
                {% else %}
                    <a href="{{ url_for('auth.login') }}" class="text-blue-500">Login</a>
                {% endif %}
            </div>
        </div>
    </nav>
    <main class="container mx-auto px-4 py-6">
        {% block content %}{% endblock %}
    </main>
</body>
</html>
```

### `app/templates/index.html`

Home page template using Shadcn components and HTMX.

```html:app/templates/index.html
{% extends "base.html" %}
{% block content %}
<div class="p-6 max-w-sm mx-auto bg-white rounded-xl shadow-md flex items-center space-x-4">
    <div>
        <div class="text-xl font-medium text-black">Welcome to My Project</div>
        <p class="text-gray-500">This is a starter template with Flask and modern frontend tools.</p>
        <button hx-get="{{ url_for('main.load_more') }}" hx-target="#more-content" class="mt-4 bg-blue-500 text-white px-4 py-2 rounded">
            Load More
        </button>
        <div id="more-content" class="mt-4"></div>
    </div>
</div>
{% endblock %}
```

### `app/routes.py`

Main routes for the application.

```python:app/routes.py
from flask import Blueprint, render_template

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
def index():
    return render_template('index.html')

@main_bp.route('/load-more')
def load_more():
    return "<p class='text-green-500'>More content loaded via HTMX!</p>"
```

### `app/static/css/styles.css`

Tailwind CSS setup.

```css:app/static/css/styles.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom styles can be added here */
```

### `tailwind.config.js`

Tailwind configuration file.

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

### `requirements.txt`

List of Python dependencies.

```
Flask
Flask-Login
Flask-OAuthlib
Flask-Migrate
Flask-SQLAlchemy
Authlib
```

### `shadcn.config.js`

Shadcn component configuration (assuming using a JS framework for Shadcn).

```javascript:shadcn.config.js
module.exports = {
  // Shadcn specific configurations
  components: [
    // List of components to include
  ],
}
```

## Additional Notes

- **Database Migrations:** The project uses Flask-Migrate to handle database migrations. Ensure you run `flask db migrate` and `flask db upgrade` when models change.

- **OAuth Providers:** The example uses Google OAuth. To add more providers, register them in `app/utils.py` using `oauth.register()`.

- **Frontend Build:** Tailwind CSS needs to be built. You can set up a build script using PostCSS or other tools to compile Tailwind styles.

- **Shadcn Integration:** Shadcn components may require additional setup depending on the framework in use. Ensure you follow Shadcn's documentation for proper integration.

- **HTMX Usage:** HTMX is used for making asynchronous requests. The example shows a simple `hx-get` usage to load more content dynamically.

## Conclusion

This template combines a robust Flask backend with a dynamic and responsive frontend using modern tools like HTMX, Tailwind CSS, and Shadcn. OAuth provides secure authentication, and SQLite offers a lightweight database solution. Customize and expand upon this foundation to suit your project's specific needs.