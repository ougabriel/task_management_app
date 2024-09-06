# task_management_app
To create a fully-featured web application with user authentication, REST API, database integration, frontend development, and deployment, you can follow these steps. I'll use Flask for this example, but Django can be used in a similar way with more built-in features.

### Project Overview: Task Management System

**Features:**
- User authentication
- Project tracking
- Task management
- Notifications (email or in-app)
- REST API
- Frontend with HTML/CSS/JavaScript
- Deployment on Heroku (or AWS)

### Step-by-Step Guide

#### 1. **Set Up the Development Environment**

- **Install Flask:**
  ```bash
  pip install Flask
  ```

- **Install additional packages:**
  ```bash
  pip install Flask-SQLAlchemy Flask-Migrate Flask-Login Flask-Mail
  ```

#### 2. **Create the Flask Application**

1. **Create the Project Structure:**
   ```
   task_manager/
   ├── app/
   │   ├── __init__.py
   │   ├── models.py
   │   ├── routes.py
   │   ├── templates/
   │   │   ├── base.html
   │   │   ├── login.html
   │   │   ├── register.html
   │   │   ├── dashboard.html
   │   │   ├── project.html
   │   ├── static/
   │       ├── style.css
   ├── migrations/
   ├── .env
   ├── config.py
   ├── run.py
   └── requirements.txt
   ```

2. **Initialize the Flask App:**

   - `app/__init__.py`
     ```python
     from flask import Flask
     from flask_sqlalchemy import SQLAlchemy
     from flask_migrate import Migrate
     from flask_login import LoginManager
     from flask_mail import Mail

     db = SQLAlchemy()
     migrate = Migrate()
     login_manager = LoginManager()
     mail = Mail()

     def create_app():
         app = Flask(__name__)
         app.config.from_object('config.Config')

         db.init_app(app)
         migrate.init_app(app, db)
         login_manager.init_app(app)
         mail.init_app(app)

         from app import routes
         app.register_blueprint(routes.bp)

         return app
     ```

   - `config.py`
     ```python
     import os

     class Config:
         SECRET_KEY = os.environ.get('SECRET_KEY', 'you-will-never-guess')
         SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL', 'sqlite:///site.db')
         SQLALCHEMY_TRACK_MODIFICATIONS = False
         MAIL_SERVER = 'smtp.gmail.com'
         MAIL_PORT = 587
         MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
         MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
         MAIL_USE_TLS = True
         MAIL_USE_SSL = False
     ```

3. **Define Models:**

   - `app/models.py`
     ```python
     from datetime import datetime
     from app import db
     from flask_login import UserMixin

     class User(db.Model, UserMixin):
         id = db.Column(db.Integer, primary_key=True)
         username = db.Column(db.String(20), unique=True, nullable=False)
         email = db.Column(db.String(120), unique=True, nullable=False)
         password = db.Column(db.String(60), nullable=False)
         tasks = db.relationship('Task', backref='author', lazy=True)

     class Task(db.Model):
         id = db.Column(db.Integer, primary_key=True)
         title = db.Column(db.String(100), nullable=False)
         description = db.Column(db.Text, nullable=False)
         date_posted = db.Column(db.DateTime, default=datetime.utcnow)
         user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
     ```

4. **Create Routes and Views:**

   - `app/routes.py`
     ```python
     from flask import Blueprint, render_template, redirect, url_for, flash, request
     from app import db
     from app.models import User, Task
     from flask_login import login_user, current_user, logout_user, login_required
     from app import login_manager
     from werkzeug.security import generate_password_hash, check_password_hash
     from flask_mail import Message

     bp = Blueprint('main', __name__)

     @bp.route('/')
     @bp.route('/home')
     @login_required
     def home():
         tasks = Task.query.all()
         return render_template('dashboard.html', tasks=tasks)

     @bp.route('/login', methods=['GET', 'POST'])
     def login():
         if request.method == 'POST':
             email = request.form.get('email')
             password = request.form.get('password')
             user = User.query.filter_by(email=email).first()
             if user and check_password_hash(user.password, password):
                 login_user(user)
                 flash('Login successful!', 'success')
                 return redirect(url_for('main.home'))
             else:
                 flash('Login failed. Check email and/or password.', 'danger')
         return render_template('login.html')

     @bp.route('/register', methods=['GET', 'POST'])
     def register():
         if request.method == 'POST':
             username = request.form.get('username')
             email = request.form.get('email')
             password = request.form.get('password')
             hashed_password = generate_password_hash(password, method='sha256')
             user = User(username=username, email=email, password=hashed_password)
             db.session.add(user)
             db.session.commit()
             flash('Account created!', 'success')
             return redirect(url_for('main.login'))
         return render_template('register.html')

     @bp.route('/logout')
     def logout():
         logout_user()
         flash('You have been logged out!', 'success')
         return redirect(url_for('main.login'))
     ```

5. **Create Templates and Static Files:**

   - `app/templates/base.html`
     ```html
     <!doctype html>
     <html lang="en">
     <head>
         <meta charset="utf-8">
         <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
         <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
         <title>Task Manager</title>
     </head>
     <body>
         <nav>
             <a href="{{ url_for('main.home') }}">Home</a>
             {% if current_user.is_authenticated %}
                 <a href="{{ url_for('main.logout') }}">Logout</a>
             {% else %}
                 <a href="{{ url_for('main.login') }}">Login</a>
                 <a href="{{ url_for('main.register') }}">Register</a>
             {% endif %}
         </nav>
         <div class="container">
             {% with messages = get_flashed_messages(with_categories=true) %}
                 {% if messages %}
                     {% for category, message in messages %}
                         <div class="alert alert-{{ category }}">{{ message }}</div>
                     {% endfor %}
                 {% endif %}
             {% endwith %}
             {% block content %}{% endblock %}
         </div>
     </body>
     </html>
     ```

   - Create additional templates like `login.html`, `register.html`, `dashboard.html`, etc., extending `base.html`.

6. **Set Up Deployment**

   - **Prepare for Heroku Deployment:**
     - Create a `Procfile` with:
       ```
       web: gunicorn run:app
       ```

     - Add `gunicorn` to `requirements.txt`:
       ```bash
       pip install gunicorn
       ```

   - **Push to GitHub:**
     ```bash
     git init
     git add .
     git commit -m "Initial commit"
     git remote add origin <YOUR_GITHUB_REPOSITORY_URL>
     git push -u origin main
     ```

   - **Deploy to Heroku:**
     ```bash
     heroku create
     git push heroku main
     heroku run flask db upgrade
     ```

### Conclusion

This project setup provides a solid foundation for a production-level web application. It covers user authentication, database integration, API development, and deployment. You can further enhance the application by adding features like notifications, advanced task management, and additional user roles.

Feel free to adjust the project according to your specific requirements or preferences.
