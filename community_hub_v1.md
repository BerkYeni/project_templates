# Flask Project Template for Sharing Twitter/X Account Lists

This Flask template provides a foundation for a platform where users can share lists of Twitter/X accounts. Other users can like and comment on these lists. The project utilizes Flask for the backend, SQLite for the database, HTMX, Tailwind CSS, and shadcn/ui for the frontend, Google OAuth 2.0 for authentication, and is designed to be hosted on a Cloudflare VPS.

## Project Structure

```plaintext
twitter_list_platform/
├── app.py
├── config.py
├── models.py
├── auth.py
├── requirements.txt
├── static/
│   ├── css/
│   │   └── styles.css
│   └── js/
│       └── main.js
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── login.html
│   ├── list_detail.html
│   └── ...
├── shadcn/
│   └── components/
│       └── ...
└── README.md
```

## Setup Instructions

1. **Clone the Repository**

   ```bash
   git clone https://github.com/yourusername/twitter_list_platform.git
   cd twitter_list_platform
   ```

2. **Create a Virtual Environment**

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

3. **Install Dependencies**

   ```bash
   pip install -r requirements.txt
   ```

4. **Configure Environment Variables**

   Create a `.env` file and set the necessary environment variables for Google OAuth and Flask configuration.

5. **Initialize the Database**

   ```bash
   flask db init
   flask db migrate
   flask db upgrade
   ```

6. **Run the Development Server**

   ```bash
   flask run
   ```

## Key Components

### 1. `app.py`

```python:app.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from auth import auth_bp
from models import db
from config import Config

def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)

    # Initialize Extensions
    db.init_app(app)
    login_manager = LoginManager()
    login_manager.init_app(app)
    login_manager.login_view = 'auth.login'

    # Register Blueprints
    app.register_blueprint(auth_bp)

    with app.app_context():
        db.create_all()

    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
```

### 2. `config.py`

```python:config.py
import os

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'your_default_secret_key'
    SQLALCHEMY_DATABASE_URI = 'sqlite:///app.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    GOOGLE_CLIENT_ID = os.environ.get('GOOGLE_CLIENT_ID')
    GOOGLE_CLIENT_SECRET = os.environ.get('GOOGLE_CLIENT_SECRET')
    OAUTHLIB_INSECURE_TRANSPORT = True  # Remove in production
```

### 3. `models.py`

```python:models.py
from flask_sqlalchemy import SQLAlchemy
from flask_login import UserMixin
from datetime import datetime

db = SQLAlchemy()

class User(UserMixin, db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    google_id = db.Column(db.String(255), unique=True, nullable=False)
    name = db.Column(db.String(255), nullable=False)
    email = db.Column(db.String(255), unique=True, nullable=False)
    lists = db.relationship('List', backref='owner', lazy=True)
    comments = db.relationship('Comment', backref='author', lazy=True)
    likes = db.relationship('Like', backref='user', lazy=True)

class List(db.Model):
    __tablename__ = 'lists'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(255), nullable=False)
    description = db.Column(db.Text, nullable=True)
    owner_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    accounts = db.relationship('Account', backref='list', lazy=True)
    comments = db.relationship('Comment', backref='list', lazy=True)
    likes = db.relationship('Like', backref='list', lazy=True)

class Account(db.Model):
    __tablename__ = 'accounts'
    
    id = db.Column(db.Integer, primary_key=True)
    twitter_handle = db.Column(db.String(255), nullable=False)
    list_id = db.Column(db.Integer, db.ForeignKey('lists.id'), nullable=False)

class Comment(db.Model):
    __tablename__ = 'comments'
    
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.Text, nullable=False)
    list_id = db.Column(db.Integer, db.ForeignKey('lists.id'), nullable=False)
    author_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class Like(db.Model):
    __tablename__ = 'likes'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    list_id = db.Column(db.Integer, db.ForeignKey('lists.id'), nullable=False)
```

### 4. `auth.py`

```python:auth.py
from flask import Blueprint, redirect, url_for, session
from flask_dance.contrib.google import make_google_blueprint, google
from models import db, User
from flask_login import login_user, logout_user, login_required

auth_bp = Blueprint('auth', __name__)

google_bp = make_google_blueprint(
    client_id='YOUR_GOOGLE_CLIENT_ID',
    client_secret='YOUR_GOOGLE_CLIENT_SECRET',
    redirect_url='/auth/google/callback',
    scope=["profile", "email"],
)

auth_bp.register_blueprint(google_bp, url_prefix="/auth")

@auth_bp.route('/login')
def login():
    if not google.authorized:
        return redirect(url_for('google.login'))
    resp = google.get("/oauth2/v2/userinfo")
    if resp.ok:
        user_info = resp.json()
        user = User.query.filter_by(email=user_info["email"]).first()
        if not user:
            user = User(
                google_id=user_info["id"],
                name=user_info["name"],
                email=user_info["email"]
            )
            db.session.add(user)
            db.session.commit()
        login_user(user)
        return redirect(url_for('main.index'))
    return redirect(url_for('auth.login'))

@auth_bp.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('main.index'))
```

### 5. `app.py` (continued with Routes)

```python:app.py
from flask import render_template, redirect, url_for, request, flash
from flask_login import LoginManager, login_required, current_user
from models import db, List, Account, Comment, Like
from auth import auth_bp
from flask import Blueprint

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
def index():
    lists = List.query.order_by(List.created_at.desc()).all()
    return render_template('index.html', lists=lists)

@main_bp.route('/list/<int:list_id>', methods=['GET', 'POST'])
def list_detail(list_id):
    list_obj = List.query.get_or_404(list_id)
    if request.method == 'POST':
        if not current_user.is_authenticated:
            flash('You need to login to comment.', 'warning')
            return redirect(url_for('auth.login'))
        content = request.form.get('content')
        if content:
            comment = Comment(content=content, list=list_obj, author=current_user)
            db.session.add(comment)
            db.session.commit()
            flash('Comment added!', 'success')
            return redirect(url_for('main.list_detail', list_id=list_id))
    return render_template('list_detail.html', list=list_obj)

@main_bp.route('/create-list', methods=['GET', 'POST'])
@login_required
def create_list():
    if request.method == 'POST':
        title = request.form.get('title')
        description = request.form.get('description')
        accounts = request.form.get('accounts').split(',')
        if title:
            new_list = List(title=title, description=description, owner=current_user)
            db.session.add(new_list)
            db.session.commit()
            for handle in accounts:
                account = Account(twitter_handle=handle.strip(), list=new_list)
                db.session.add(account)
            db.session.commit()
            flash('List created successfully!', 'success')
            return redirect(url_for('main.index'))
    return render_template('create_list.html')

@main_bp.route('/like/<int:list_id>', methods=['POST'])
@login_required
def like_list(list_id):
    list_obj = List.query.get_or_404(list_id)
    existing_like = Like.query.filter_by(user_id=current_user.id, list_id=list_id).first()
    if existing_like:
        db.session.delete(existing_like)
        db.session.commit()
        return {'status': 'unliked', 'total_likes': len(list_obj.likes)}
    else:
        like = Like(user=current_user, list=list_obj)
        db.session.add(like)
        db.session.commit()
        return {'status': 'liked', 'total_likes': len(list_obj.likes)}

# Register Main Blueprint
app = create_app()
app.register_blueprint(main_bp)

# Flask-Login User Loader
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))
```

### 6. `requirements.txt`

```plaintext
Flask
Flask-Dance
Flask-Login
Flask-SQLAlchemy
HTMX
tailwindcss
shadcn-ui
```

### 7. `templates/base.html`

```html:templates/base.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Twitter List Platform{% endblock %}</title>
    <link href="{{ url_for('static', filename='css/styles.css') }}" rel="stylesheet">
    <script src="https://unpkg.com/htmx.org@1.9.2"></script>
</head>
<body class="bg-gray-100">
    <nav class="bg-white shadow">
        <div class="max-w-7xl mx-auto px-4">
            <div class="flex justify-between">
                <div class="flex space-x-4">
                    <a href="{{ url_for('main.index') }}" class="flex items-center py-5 px-2 text-gray-700">Home</a>
                    <a href="{{ url_for('main.create_list') }}" class="flex items-center py-5 px-2 text-gray-700">Create List</a>
                </div>
                <div>
                    {% if current_user.is_authenticated %}
                        <span class="text-gray-700">Hello, {{ current_user.name }}</span>
                        <a href="{{ url_for('auth.logout') }}" class="text-gray-700 ml-4">Logout</a>
                    {% else %}
                        <a href="{{ url_for('auth.login') }}" class="text-gray-700">Login</a>
                    {% endif %}
                </div>
            </div>
        </div>
    </nav>

    <div class="container mx-auto mt-4">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="mb-4 p-4 border rounded {{ 'bg-green-100 border-green-400 text-green-700' if category == 'success' else 'bg-yellow-100 border-yellow-400 text-yellow-700' }}">
                        {{ message }}
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </div>

    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
</body>
</html>
```

### 8. `templates/index.html`

```html:templates/index.html
{% extends 'base.html' %}

{% block title %}Home - Twitter List Platform{% endblock %}

{% block content %}
<h1 class="text-2xl font-bold mb-4">Shared Twitter/X Lists</h1>
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    {% for list in lists %}
    <div class="bg-white p-4 rounded shadow">
        <h2 class="text-xl font-semibold">{{ list.title }}</h2>
        <p class="text-gray-600">{{ list.description }}</p>
        <p class="text-sm text-gray-500">By {{ list.owner.name }} on {{ list.created_at.strftime('%Y-%m-%d') }}</p>
        <div class="mt-2 flex items-center">
            <button hx-post="{{ url_for('main.like_list', list_id=list.id) }}" hx-swap="outerHTML" class="text-blue-500">
                {% if current_user.is_authenticated and list in current_user.likes %}
                    Unlike
                {% else %}
                    Like
                {% endif %}
            </button>
            <span class="ml-2">{{ list.likes|length }} Likes</span>
        </div>
        <a href="{{ url_for('main.list_detail', list_id=list.id) }}" class="text-blue-700 mt-2 inline-block">View Details</a>
    </div>
    {% endfor %}
</div>
{% endblock %}
```

### 9. `templates/list_detail.html`

```html:templates/list_detail.html
{% extends 'base.html' %}

{% block title %}{{ list.title }} - Twitter List Platform{% endblock %}

{% block content %}
<div class="bg-white p-6 rounded shadow">
    <h1 class="text-3xl font-bold">{{ list.title }}</h1>
    <p class="text-gray-700 mt-2">{{ list.description }}</p>
    <p class="text-sm text-gray-500">By {{ list.owner.name }} on {{ list.created_at.strftime('%Y-%m-%d') }}</p>
    
    <h2 class="text-2xl font-semibold mt-4">Accounts</h2>
    <ul class="list-disc list-inside">
        {% for account in list.accounts %}
            <li><a href="https://twitter.com/{{ account.twitter_handle }}" target="_blank" class="text-blue-500">@{{ account.twitter_handle }}</a></li>
        {% endfor %}
    </ul>

    <div class="mt-4">
        <form method="POST" class="flex flex-col">
            <label for="content" class="font-medium">Add a Comment:</label>
            <textarea name="content" id="content" rows="3" class="border rounded p-2 mt-1"></textarea>
            <button type="submit" class="mt-2 bg-blue-500 text-white px-4 py-2 rounded">Submit</button>
        </form>
    </div>

    <div class="mt-4">
        <h3 class="text-xl font-semibold">Comments</h3>
        {% for comment in list.comments %}
            <div class="border-b py-2">
                <p class="text-gray-800">{{ comment.content }}</p>
                <p class="text-sm text-gray-500">By {{ comment.author.name }} on {{ comment.created_at.strftime('%Y-%m-%d') }}</p>
            </div>
        {% endfor %}
    </div>
</div>
{% endblock %}
```

### 10. `static/css/styles.css`

Configure Tailwind CSS by initializing it and adding the necessary directives.

```css:static/css/styles.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom styles can be added here */
```

### 11. `static/js/main.js`

Initialize shadcn/ui components and handle any custom JavaScript if needed.

```javascript:static/js/main.js
// Initialize shadcn/UI components if necessary
// Example: Initialize modals, dropdowns, etc.

// Example HTMX event listeners
document.body.addEventListener('htmx:afterSwap', (event) => {
    // Handle events after HTMX swaps content
});
```

### 12. `shadcn/components/` 

Integrate shadcn/UI components as per your design requirements. You can customize or create new components based on the [shadcn/ui documentation](https://shadcn.com/).

## Deployment with Cloudflare VPS

1. **Set Up Your VPS**

   - Provision a VPS with your preferred operating system.
   - Install necessary packages like Python, pip, Nginx, etc.

2. **Clone the Repository on the VPS**

   ```bash
   git clone https://github.com/yourusername/twitter_list_platform.git
   cd twitter_list_platform
   ```

3. **Set Up the Virtual Environment and Install Dependencies**

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

4. **Configure Environment Variables**

   Set up environment variables securely, possibly using a `.env` file or exporting them in your shell.

5. **Configure Nginx as a Reverse Proxy**

   Set up Nginx to serve your Flask application. Here's a basic configuration:

   ```nginx
   server {
       listen 80;
       server_name yourdomain.com;

       location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

       location /static/ {
           alias /path/to/twitter_list_platform/static/;
       }
   }
   ```

6. **Run the Flask Application**

   Use a production server like Gunicorn:

   ```bash
   gunicorn app:app -b 127.0.0.1:8000
   ```

7. **Secure Your Application**

   - Obtain SSL certificates, possibly using Cloudflare's SSL features.
   - Ensure environment variables are securely stored.
   - Regularly update dependencies and monitor for security patches.

## Additional Notes

- **HTMX Integration:** HTMX allows you to create dynamic user experiences with minimal JavaScript. You can enhance forms and interactions by adding `hx-*` attributes in your HTML templates.

- **Tailwind CSS:** Tailwind provides utility-first CSS classes to build responsive and modern interfaces. Customize your `tailwind.config.js` as needed.

- **Shadcn/UI Components:** Utilize shadcn's pre-built components to accelerate frontend development. Ensure you import and initialize them correctly in your project.

- **Google OAuth 2.0:** Replace `'YOUR_GOOGLE_CLIENT_ID'` and `'YOUR_GOOGLE_CLIENT_SECRET'` with your actual credentials obtained from the Google Developer Console.

- **Database Migrations:** Consider using Flask-Migrate for handling database migrations more efficiently.

- **Error Handling & Security:** Implement proper error handling, input validation, and security best practices to protect your application.

This template serves as a starting point. Customize and expand it based on your project's specific requirements and design preferences.