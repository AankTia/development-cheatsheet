# Flask CRUD Web Application with Advanced Features

Building a comprehensive Flask web application. This will be a step-by-step approach covering everything from setup to deployment.

This comprehensive Flask application includes:
- User authentication (login, registration, password reset)
- Input validation (forms with WTForms)
- Pagination (Flask-SQLAlchemy paginate)
- Search functionality
- Testing (pytest)
- Role-based authorization (admin vs regular users)
- File uploads (with image processing)
- API endpoints (JSON responses)
- Real-time updates (SocketIO)
- Local development setup
- Docker configuration
- Deployment to Heroku and Render
- AWS deployment would follow similar patterns with appropriate configuration for RDS, S3, etc.

## Project Structure

```
flask_crud_app/
├── app/
│   ├── __init__.py
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── forms.py
│   │   ├── routes.py
│   │   └── utils.py
│   ├── main/
│   │   ├── __init__.py
│   │   ├── forms.py
│   │   ├── routes.py
│   │   └── utils.py
│   ├── models.py
│   ├── static/
│   │   ├── css/
│   │   ├── js/
│   │   └── uploads/
│   ├── templates/
│   │   ├── auth/
│   │   ├── base.html
│   │   ├── errors/
│   │   └── main/
│   └── utils.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_auth.py
│   └── test_main.py
├── migrations/
├── requirements.txt
├── config.py
├── docker-compose.yml
├── Dockerfile
├── Procfile
├── runtime.txt
└── app.py
```

## Step 1: Setup and Basic Configuration

### Install required packages
```bash
pip install flask flask-sqlalchemy flask-login flask-migrate flask-wtf flask-bootstrap flask-importmap python-dotenv flask-limiter flask-caching
```

### config.py
```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'your-secret-key-here'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'sqlite:///site.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    UPLOAD_FOLDER = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'app', 'static', 'uploads')
    ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif', 'pdf'}
    MAIL_SERVER = 'smtp.googlemail.com'
    MAIL_PORT = 587
    MAIL_USE_TLS = True
    MAIL_USERNAME = os.environ.get('EMAIL_USER')
    MAIL_PASSWORD = os.environ.get('EMAIL_PASS')
    POSTS_PER_PAGE = 10
    CACHE_TYPE = 'SimpleCache'
    CACHE_DEFAULT_TIMEOUT = 300
```

### app/__init__.py
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_migrate import Migrate
from flask_bootstrap import Bootstrap5
from flask_importmap import Importmap
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from flask_caching import Cache
from config import Config

db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()
login_manager.login_view = 'auth.login'
login_manager.login_message_category = 'info'
bootstrap = Bootstrap5()
importmap = Importmap()
cache = Cache()

limiter = Limiter(
    get_remote_address,
    default_limits=["200 per day", "50 per hour"],
    storage_uri="memory://",
)

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)
    bootstrap.init_app(app)
    importmap.init_app(app)
    limiter.init_app(app)
    cache.init_app(app)

    from app.auth import bp as auth_bp
    app.register_blueprint(auth_bp, url_prefix='/auth')

    from app.main import bp as main_bp
    app.register_blueprint(main_bp)

    @app.shell_context_processor
    def make_shell_context():
        return {'db': db}

    return app

from app import models
```

## Step 2: Database Models

### app/models.py
```python
from datetime import datetime
from app import db, login_manager
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), index=True, unique=True, nullable=False)
    email = db.Column(db.String(120), index=True, unique=True, nullable=False)
    password_hash = db.Column(db.String(128))
    role = db.Column(db.String(20), default='user')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_seen = db.Column(db.DateTime, default=datetime.utcnow)
    items = db.relationship('Item', backref='author', lazy='dynamic')

    def __repr__(self):
        return f'<User {self.username}>'

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

    def is_admin(self):
        return self.role == 'admin'

class Item(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text)
    image_file = db.Column(db.String(20))
    created_at = db.Column(db.DateTime, index=True, default=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

    def __repr__(self):
        return f'<Item {self.title}>'

@login_manager.user_loader
def load_user(id):
    return User.query.get(int(id))
```

## Step 3: Authentication System

### app/auth/forms.py
```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField, BooleanField
from wtforms.validators import DataRequired, Length, Email, EqualTo, ValidationError
from app.models import User

class RegistrationForm(FlaskForm):
    username = StringField('Username', 
                          validators=[DataRequired(), Length(min=2, max=20)])
    email = StringField('Email',
                       validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired()])
    confirm_password = PasswordField('Confirm Password',
                                    validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Sign Up')

    def validate_username(self, username):
        user = User.query.filter_by(username=username.data).first()
        if user:
            raise ValidationError('That username is taken. Please choose a different one.')

    def validate_email(self, email):
        user = User.query.filter_by(email=email.data).first()
        if user:
            raise ValidationError('That email is taken. Please choose a different one.')

class LoginForm(FlaskForm):
    email = StringField('Email',
                       validators=[DataRequired(), Email()])
    password = PasswordField('Password', validators=[DataRequired()])
    remember = BooleanField('Remember Me')
    submit = SubmitField('Login')

class RequestResetForm(FlaskForm):
    email = StringField('Email',
                        validators=[DataRequired(), Email()])
    submit = SubmitField('Request Password Reset')

    def validate_email(self, email):
        user = User.query.filter_by(email=email.data).first()
        if user is None:
            raise ValidationError('There is no account with that email. You must register first.')

class ResetPasswordForm(FlaskForm):
    password = PasswordField('Password', validators=[DataRequired()])
    confirm_password = PasswordField('Confirm Password',
                                    validators=[DataRequired(), EqualTo('password')])
    submit = SubmitField('Reset Password')
```

### app/auth/routes.py
```python
from flask import render_template, url_for, flash, redirect, request, Blueprint
from flask_login import login_user, current_user, logout_user, login_required
from app import db, bcrypt
from app.models import User
from app.auth.forms import RegistrationForm, LoginForm, RequestResetForm, ResetPasswordForm
from app.auth.utils import send_reset_email

bp = Blueprint('auth', __name__)

@bp.route('/register', methods=['GET', 'POST'])
def register():
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    form = RegistrationForm()
    if form.validate_on_submit():
        hashed_password = bcrypt.generate_password_hash(form.password.data).decode('utf-8')
        user = User(username=form.username.data, email=form.email.data, password=hashed_password)
        db.session.add(user)
        db.session.commit()
        flash('Your account has been created! You are now able to log in', 'success')
        return redirect(url_for('auth.login'))
    return render_template('auth/register.html', title='Register', form=form)

@bp.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if user and bcrypt.check_password_hash(user.password_hash, form.password.data):
            login_user(user, remember=form.remember.data)
            next_page = request.args.get('next')
            return redirect(next_page) if next_page else redirect(url_for('main.index'))
        else:
            flash('Login Unsuccessful. Please check email and password', 'danger')
    return render_template('auth/login.html', title='Login', form=form)

@bp.route('/logout')
def logout():
    logout_user()
    return redirect(url_for('main.index'))

@bp.route('/reset_password', methods=['GET', 'POST'])
def reset_request():
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    form = RequestResetForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        send_reset_email(user)
        flash('An email has been sent with instructions to reset your password.', 'info')
        return redirect(url_for('auth.login'))
    return render_template('auth/reset_request.html', title='Reset Password', form=form)

@bp.route('/reset_password/<token>', methods=['GET', 'POST'])
def reset_token(token):
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    user = User.verify_reset_token(token)
    if user is None:
        flash('That is an invalid or expired token', 'warning')
        return redirect(url_for('auth.reset_request'))
    form = ResetPasswordForm()
    if form.validate_on_submit():
        hashed_password = bcrypt.generate_password_hash(form.password.data).decode('utf-8')
        user.password = hashed_password
        db.session.commit()
        flash('Your password has been updated! You are now able to log in', 'success')
        return redirect(url_for('auth.login'))
    return render_template('auth/reset_token.html', title='Reset Password', form=form)
```

## Step 4: Main Application Functionality

### app/main/forms.py
```python
from flask_wtf import FlaskForm
from flask_wtf.file import FileField, FileAllowed
from wtforms import StringField, TextAreaField, SubmitField
from wtforms.validators import DataRequired, Length

class ItemForm(FlaskForm):
    title = StringField('Title', validators=[DataRequired(), Length(max=100)])
    description = TextAreaField('Description')
    image = FileField('Upload Image', validators=[FileAllowed(['jpg', 'png', 'jpeg', 'gif'])])
    submit = SubmitField('Submit')

class SearchForm(FlaskForm):
    q = StringField('Search', validators=[DataRequired()])
```

### app/main/routes.py
```python
from flask import render_template, url_for, flash, redirect, request, abort, jsonify
from flask_login import current_user, login_required
from app import db, cache
from app.models import User, Item
from app.main.forms import ItemForm, SearchForm
from app.main.utils import save_picture
from datetime import datetime
import os

bp = Blueprint('main', __name__)

@bp.before_app_request
def before_request():
    if current_user.is_authenticated:
        current_user.last_seen = datetime.utcnow()
        db.session.commit()

@bp.route('/', methods=['GET', 'POST'])
@bp.route('/home', methods=['GET', 'POST'])
@cache.cached(timeout=300)
def index():
    form = SearchForm()
    page = request.args.get('page', 1, type=int)
    
    if form.validate_on_submit():
        return redirect(url_for('main.search', query=form.q.data))
    
    items = Item.query.order_by(Item.created_at.desc()).paginate(page=page, per_page=current_app.config['POSTS_PER_PAGE'])
    return render_template('main/home.html', items=items, form=form)

@bp.route('/search')
@cache.cached(timeout=300, query_string=True)
def search():
    query = request.args.get('query', '')
    page = request.args.get('page', 1, type=int)
    
    items = Item.query.filter(
        Item.title.ilike(f'%{query}%') | 
        Item.description.ilike(f'%{query}%')
    ).paginate(page=page, per_page=current_app.config['POSTS_PER_PAGE'])
    
    return render_template('main/search_results.html', items=items, query=query)

@bp.route('/item/new', methods=['GET', 'POST'])
@login_required
def new_item():
    form = ItemForm()
    if form.validate_on_submit():
        image_file = None
        if form.image.data:
            image_file = save_picture(form.image.data)
            
        item = Item(
            title=form.title.data,
            description=form.description.data,
            image_file=image_file,
            author=current_user
        )
        db.session.add(item)
        db.session.commit()
        flash('Your item has been created!', 'success')
        return redirect(url_for('main.index'))
    return render_template('main/create_item.html', title='New Item', form=form)

@bp.route('/item/<int:item_id>')
@cache.cached(timeout=300)
def item(item_id):
    item = Item.query.get_or_404(item_id)
    return render_template('main/item.html', title=item.title, item=item)

@bp.route('/item/<int:item_id>/update', methods=['GET', 'POST'])
@login_required
def update_item(item_id):
    item = Item.query.get_or_404(item_id)
    if item.author != current_user and not current_user.is_admin():
        abort(403)
    form = ItemForm()
    if form.validate_on_submit():
        if form.image.data:
            if item.image_file:
                try:
                    os.remove(os.path.join(current_app.config['UPLOAD_FOLDER'], item.image_file))
                except FileNotFoundError:
                    pass
            item.image_file = save_picture(form.image.data)
            
        item.title = form.title.data
        item.description = form.description.data
        db.session.commit()
        flash('Your item has been updated!', 'success')
        return redirect(url_for('main.item', item_id=item.id))
    elif request.method == 'GET':
        form.title.data = item.title
        form.description.data = item.description
    return render_template('main/create_item.html', title='Update Item', form=form)

@bp.route('/item/<int:item_id>/delete', methods=['POST'])
@login_required
def delete_item(item_id):
    item = Item.query.get_or_404(item_id)
    if item.author != current_user and not current_user.is_admin():
        abort(403)
    
    if item.image_file:
        try:
            os.remove(os.path.join(current_app.config['UPLOAD_FOLDER'], item.image_file))
        except FileNotFoundError:
            pass
    
    db.session.delete(item)
    db.session.commit()
    flash('Your item has been deleted!', 'success')
    return redirect(url_for('main.index'))

# API Endpoints
@bp.route('/api/items', methods=['GET'])
def api_get_items():
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 10, type=int)
    items = Item.query.order_by(Item.created_at.desc()).paginate(page=page, per_page=per_page)
    
    return jsonify({
        'items': [{
            'id': item.id,
            'title': item.title,
            'description': item.description,
            'image_file': item.image_file,
            'created_at': item.created_at,
            'author': item.author.username
        } for item in items.items],
        'total': items.total,
        'pages': items.pages,
        'current_page': items.page
    })

@bp.route('/api/items/<int:item_id>', methods=['GET'])
def api_get_item(item_id):
    item = Item.query.get_or_404(item_id)
    return jsonify({
        'id': item.id,
        'title': item.title,
        'description': item.description,
        'image_file': item.image_file,
        'created_at': item.created_at,
        'author': item.author.username
    })
```

## Step 5: Utility Functions

### app/main/utils.py
```python
import os
import secrets
from PIL import Image
from flask import current_app

def save_picture(form_picture):
    random_hex = secrets.token_hex(8)
    _, f_ext = os.path.splitext(form_picture.filename)
    picture_fn = random_hex + f_ext
    picture_path = os.path.join(current_app.config['UPLOAD_FOLDER'], picture_fn)
    
    output_size = (500, 500)
    i = Image.open(form_picture)
    i.thumbnail(output_size)
    i.save(picture_path)
    
    return picture_fn
```

### app/auth/utils.py
```python
from itsdangerous import URLSafeTimedSerializer as Serializer
from flask import current_app
from app.models import User

def get_reset_token(user, expires_sec=1800):
    s = Serializer(current_app.config['SECRET_KEY'], expires_sec)
    return s.dumps({'user_id': user.id}).decode('utf-8')

def verify_reset_token(token):
    s = Serializer(current_app.config['SECRET_KEY'])
    try:
        user_id = s.loads(token)['user_id']
    except:
        return None
    return User.query.get(user_id)

def send_reset_email(user):
    token = get_reset_token(user)
    # Implement email sending logic here
    pass
```

## Step 6: Templates

### Base Template (app/templates/base.html)
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}{% endblock %} - Flask CRUD App</title>
    {{ importmap.import("bootstrap") }}
    <link rel="stylesheet" href="{{ url_for('static', filename='css/styles.css') }}">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('main.index') }}">Flask CRUD</a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.index') }}">Home</a>
                    </li>
                    {% if current_user.is_authenticated %}
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.new_item') }}">New Item</a>
                    </li>
                    {% endif %}
                </ul>
                <ul class="navbar-nav">
                    {% if current_user.is_authenticated %}
                    <li class="nav-item dropdown">
                        <a class="nav-link dropdown-toggle" href="#" id="navbarDropdown" role="button" data-bs-toggle="dropdown">
                            {{ current_user.username }}
                        </a>
                        <ul class="dropdown-menu">
                            <li><a class="dropdown-item" href="#">Profile</a></li>
                            <li><hr class="dropdown-divider"></li>
                            <li><a class="dropdown-item" href="{{ url_for('auth.logout') }}">Logout</a></li>
                        </ul>
                    </li>
                    {% else %}
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('auth.login') }}">Login</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('auth.register') }}">Register</a>
                    </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }} alert-dismissible fade show" role="alert">
                        {{ message }}
                        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        {% block content %}{% endblock %}
    </div>

    <footer class="mt-5 py-3 bg-light">
        <div class="container text-center">
            <span class="text-muted">Flask CRUD App © 2023</span>
        </div>
    </footer>

    {{ importmap.import("bootstrap") }}
    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
    {% block scripts %}{% endblock %}
</body>
</html>
```

### Home Template (app/templates/main/home.html)
```html
{% extends "base.html" %}
{% block content %}
    <h1 class="mb-4">Latest Items</h1>
    
    <form class="mb-4" method="GET" action="{{ url_for('main.search') }}">
        <div class="input-group">
            {{ form.q(class="form-control", placeholder="Search items...") }}
            <button class="btn btn-outline-secondary" type="submit">Search</button>
        </div>
    </form>
    
    {% for item in items.items %}
        <article class="media content-section mb-4">
            <div class="media-body">
                <div class="article-metadata">
                    <a class="me-2" href="#">{{ item.author.username }}</a>
                    <small class="text-muted">{{ item.created_at.strftime('%Y-%m-%d') }}</small>
                </div>
                <h2><a class="article-title" href="{{ url_for('main.item', item_id=item.id) }}">{{ item.title }}</a></h2>
                <p class="article-content">{{ item.description }}</p>
                {% if item.image_file %}
                    <img src="{{ url_for('static', filename='uploads/' + item.image_file) }}" class="img-fluid mb-3" style="max-height: 300px;">
                {% endif %}
            </div>
        </article>
    {% endfor %}
    
    <nav aria-label="Page navigation">
        <ul class="pagination">
            {% if items.has_prev %}
                <li class="page-item">
                    <a class="page-link" href="{{ url_for('main.index', page=items.prev_num) }}">Previous</a>
                </li>
            {% else %}
                <li class="page-item disabled">
                    <span class="page-link">Previous</span>
                </li>
            {% endif %}
            
            {% for page_num in items.iter_pages(left_edge=1, right_edge=1, left_current=1, right_current=2) %}
                {% if page_num %}
                    {% if items.page == page_num %}
                        <li class="page-item active">
                            <span class="page-link">{{ page_num }}</span>
                        </li>
                    {% else %}
                        <li class="page-item">
                            <a class="page-link" href="{{ url_for('main.index', page=page_num) }}">{{ page_num }}</a>
                        </li>
                    {% endif %}
                {% else %}
                    <li class="page-item disabled">
                        <span class="page-link">...</span>
                    </li>
                {% endif %}
            {% endfor %}
            
            {% if items.has_next %}
                <li class="page-item">
                    <a class="page-link" href="{{ url_for('main.index', page=items.next_num) }}">Next</a>
                </li>
            {% else %}
                <li class="page-item disabled">
                    <span class="page-link">Next</span>
                </li>
            {% endif %}
        </ul>
    </nav>
{% endblock content %}
```

## Step 7: Testing

### tests/conftest.py
```python
import pytest
from app import create_app, db
from app.models import User, Item
from config import Config

class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = 'sqlite://'
    WTF_CSRF_ENABLED = False

@pytest.fixture
def app():
    app = create_app(TestConfig)
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def init_database(app):
    with app.app_context():
        # Create test user
        user = User(username='testuser', email='test@example.com')
        user.set_password('testpassword')
        db.session.add(user)
        
        # Create test admin
        admin = User(username='admin', email='admin@example.com', role='admin')
        admin.set_password('adminpassword')
        db.session.add(admin)
        
        # Create test items
        item1 = Item(title='Test Item 1', description='Description 1', author=user)
        item2 = Item(title='Test Item 2', description='Description 2', author=admin)
        db.session.add_all([item1, item2])
        
        db.session.commit()
    
    yield
    
    with app.app_context():
        db.drop_all()
```

### tests/test_auth.py
```python
def test_register(client, init_database):
    response = client.post('/auth/register', data={
        'username': 'newuser',
        'email': 'new@example.com',
        'password': 'newpassword',
        'confirm_password': 'newpassword'
    }, follow_redirects=True)
    assert b'Your account has been created!' in response.data

def test_login(client, init_database):
    response = client.post('/auth/login', data={
        'email': 'test@example.com',
        'password': 'testpassword',
        'remember': False
    }, follow_redirects=True)
    assert b'Login Unsuccessful' not in response.data
```

### tests/test_main.py
```python
def test_home_page(client, init_database):
    response = client.get('/')
    assert response.status_code == 200
    assert b'Test Item 1' in response.data

def test_create_item_authenticated(client, init_database):
    # Login first
    client.post('/auth/login', data={
        'email': 'test@example.com',
        'password': 'testpassword',
        'remember': False
    }, follow_redirects=True)
    
    response = client.post('/item/new', data={
        'title': 'New Test Item',
        'description': 'New Description',
    }, follow_redirects=True)
    assert b'Your item has been created!' in response.data
```

## Step 8: Docker Configuration

### Dockerfile
```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . .

# Run the application
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:create_app()"]
```

### docker-compose.yml
```yaml
version: '3.8'

services:
  web:
    build: .
    command: flask run --host=0.0.0.0 --port=8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    environment:
      - FLASK_APP=app.py
      - FLASK_ENV=development
      - DATABASE_URL=sqlite:////app/instance/site.db
      - SECRET_KEY=your-secret-key-here
    depends_on:
      - db

  db:
    image: postgres:13
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_USER=flaskuser
      - POSTGRES_PASSWORD=flaskpass
      - POSTGRES_DB=flaskdb

volumes:
  postgres_data:
```

## Step 9: Deployment Configuration

### Procfile (for Heroku)
```
web: gunicorn app:create_app()
```

### runtime.txt (for Heroku)
```
python-3.9.16
```

## Step 10: Real-time Updates with Flask-SocketIO

Add to requirements.txt:
```
flask-socketio
```

### app/__init__.py (add these lines)
```python
from flask_socketio import SocketIO

socketio = SocketIO()

def create_app(config_class=Config):
    # ... existing code ...
    socketio.init_app(app)
    return app
```

### app/main/routes.py (add this route)
```python
from flask_socketio import emit

@socketio.on('connect')
def handle_connect():
    emit('message', {'data': 'Connected'})

@socketio.on('new_item')
def handle_new_item(data):
    emit('new_item', data, broadcast=True)
```

### Update base.html to include SocketIO
```html
<!-- Add before closing body tag -->
{{ importmap.import("socket.io-client") }}
<script>
    const socket = io();
    socket.on('new_item', function(data) {
        // Handle real-time updates
        console.log('New item created:', data);
        // You could refresh the page or update the DOM dynamically
        location.reload();
    });
</script>
```

## Final Steps

1. Initialize the database:
```bash
flask db init
flask db migrate -m "initial migration"
flask db upgrade
```

2. Create an admin user (run in Flask shell):
```python
from app.models import User
user = User(username='admin', email='admin@example.com', role='admin')
user.set_password('adminpassword')
db.session.add(user)
db.session.commit()
```

3. Run the application locally:
```bash
flask run
```

4. Build and run with Docker:
```bash
docker-compose up --build
```

5. Deploy to Heroku:
```bash
heroku create
git push heroku main
heroku run flask db upgrade
```

6. Deploy to Render:
   - Connect your GitHub repository
   - Set environment variables (DATABASE_URL, SECRET_KEY, etc.)
   - Specify the build command and start command
