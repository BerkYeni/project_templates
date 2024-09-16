# Flask Project Template

## File Structure

```
project/
├── app.py
├── requirements.txt
├── templates/
│   ├── base.html
│   └── index.html
├── static/
│   └── css/
│       └── styles.css
└── notes.md
```

## app.py

```python
from flask import Flask, render_template
import sqlite3

app = Flask(__name__)

def get_db_connection():
    conn = sqlite3.connect('database.db')
    conn.row_factory = sqlite3.Row
    return conn

@app.route('/')
def index():
    conn = get_db_connection()
    # Example query
    # rows = conn.execute('SELECT * FROM table').fetchall()
    conn.close()
    return render_template('index.html')

if __name__ == '__main__':
    app.run(debug=True)
```

## requirements.txt

```
Flask
htmx
```

## templates/base.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Flask App</title>
    <link href="/static/css/styles.css" rel="stylesheet">
    <script src="https://unpkg.com/htmx.org@1.9.2"></script>
</head>
<body class="bg-gray-100">
    {% block content %}{% endblock %}
</body>
</html>
```

## templates/index.html

```html
{% extends 'base.html' %}

{% block content %}
<div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold">Welcome to Flask with HTMX and Tailwind!</h1>
    <!-- Your content here -->
</div>
{% endblock %}
```

## static/css/styles.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## notes.md

# Project Setup

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd project
   ```

2. **Create a virtual environment and activate it**

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

3. **Install dependencies**

   ```bash
   pip install -r requirements.txt
   ```

4. **Set up the database**

   ```bash
   # In Python shell
   from app import get_db_connection
   conn = get_db_connection()
   # Execute SQL commands to create tables
   conn.close()
   ```

5. **Compile Tailwind CSS**

   Ensure Tailwind CSS is set up. You can use the CDN as included in `base.html` or set up a build process for custom styles.

6. **Run the application**

   ```bash
   python app.py
   ```

# Notes

- **Flask** is used as the backend framework.
- **SQLite** serves as the database for simplicity.
- **HTMX** enables dynamic interactions without full page reloads.
- **Tailwind CSS** provides utility-first styling.
- Adjust `app.py` and templates as needed to fit your project requirements.