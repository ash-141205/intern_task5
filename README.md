# intern_task5
from flask import Flask, render_template, request, redirect, session, url_for
from werkzeug.utils import secure_filename
import os

from models import db, User, Post
from datetime import datetime

# 🔧 Flask app configuration
app = Flask(__name__)
app.secret_key = "supersecretkey"

# 📁 Use absolute path for database file
basedir = os.path.abspath(os.path.dirname(__file__))
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'instance', 'social.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['UPLOAD_FOLDER'] = os.path.join('static', 'uploads')

# 📦 Initialize DB
db.init_app(app)

with app.app_context():
    # Ensure folders exist
    os.makedirs(os.path.join(basedir, 'instance'), exist_ok=True)
    os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)
    db.create_all()

# 🏠 Homepage (feed)
@app.route('/')
def home():
    if 'user_id' not in session:
        return redirect('/login')
    posts = Post.query.order_by(Post.timestamp.desc()).all()
    return render_template('feed.html', posts=posts)

# 📝 Register
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        if User.query.filter_by(username=username).first():
            return "User already exists."
        user = User(username=username, password=password)
        db.session.add(user)
        db.session.commit()
        return redirect('/login')
    return render_template('register.html')

# 🔑 Login
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        user = User.query.filter_by(username=username, password=password).first()
        if user:
            session['user_id'] = user.id
            return redirect('/')
        return "Invalid username or password"
    return render_template('login.html')

# 🚪 Logout
@app.route('/logout')
def logout():
    session.clear()
    return redirect('/login')

# 🖼️ Post creation
@app.route('/create_post', methods=['POST'])
def create_post():
    if 'user_id' not in session:
        return redirect('/login')
    content = request.form['content']
    file = request.files.get('media')
    filename = None
    if file and file.filename:
        filename = secure_filename(file.filename)
        file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
    post = Post(user_id=session['user_id'], content=content, media=filename)
    db.session.add(post)
    db.session.commit()
    return redirect('/')

# 🚀 Run the app
if __name__ == '__main__':
    app.run(debug=True)
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

db = SQLAlchemy()

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(100), unique=True, nullable=False)
    password = db.Column(db.String(100), nullable=False)
    profile_pic = db.Column(db.String(100), default='default.png')  # Optional

class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    content = db.Column(db.Text, nullable=False)
    media = db.Column(db.String(100))
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)

    user = db.relationship('User', backref='posts')

#HTML(TEMP)
<!DOCTYPE html>
<html>
<head>
    <title>Feed</title>
</head>
<body>
    <h2>Welcome to Social Feed</h2>
    <p><a href="/logout">Logout</a></p>

    <form method="POST" action="/create_post" enctype="multipart/form-data">
        <textarea name="content" placeholder="What's on your mind?" required></textarea><br>
        <input type="file" name="media"><br><br>
        <input type="submit" value="Post">
    </form>

    <hr>
    {% for post in posts %}
        <div style="border: 1px solid #ccc; padding: 10px; margin: 10px 0;">
            <p><strong>{{ post.user.username }}</strong> - {{ post.timestamp.strftime('%Y-%m-%d %H:%M') }}</p>
            <p>{{ post.content }}</p>
            {% if post.media %}
                <img src="{{ url_for('static', filename='uploads/' + post.media) }}" width="300">
            {% endif %}
        </div>
    {% endfor %}
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <title>Login</title>
</head>
<body>
    <h2>Login</h2>
    <form method="POST">
        <label>Username:</label>
        <input type="text" name="username" required><br><br>
        <label>Password:</label>
        <input type="password" name="password" required><br><br>
        <input type="submit" value="Login">
    </form>
    <p>Don't have an account? <a href="/register">Register here</a></p>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <title>Register</title>
</head>
<body>
    <h2>Register</h2>
    <form method="POST">
        <label>Username:</label>
        <input type="text" name="username" required><br><br>
        <label>Password:</label>
        <input type="password" name="password" required><br><br>
        <input type="submit" value="Register">
    </form>
    <p>Already have an account? <a href="/login">Login here</a></p>
</body>
</html>

