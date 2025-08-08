FILE: app.py

""" A minimal Flask backend + simple frontend for a cloud-deployable lifelong-learning app. Includes:

User signup/login (simple, not production-ready)

Courses listing and enrollment

Progress tracking (in SQLite for simplicity)

REST endpoints for a single-page frontend


To run locally:

1. python -m venv venv


2. source venv/bin/activate   (Linux/Mac) or venv\Scripts\activate (Windows)


3. pip install -r requirements.txt


4. flask run



To deploy to a container-based cloud (Cloud Run, Render, Railway):

Build Docker image using the provided Dockerfile

Push to container registry and deploy


This is intentionally simple and meant for learning. Do NOT use this exact auth or secret handling in production. """ from flask import Flask, request, jsonify, render_template, g import sqlite3 import os from werkzeug.security import generate_password_hash, check_password_hash

DATABASE = os.path.join(os.path.dirname(file), 'app.db') app = Flask(name) app.config['SECRET_KEY'] = 'dev-secret-key-change-me'

---------- Database helpers ----------

def get_db(): db = getattr(g, '_database', None) if db is None: db = g._database = sqlite3.connect(DATABASE) db.row_factory = sqlite3.Row return db

@app.teardown_appcontext def close_connection(exception): db = getattr(g, '_database', None) if db is not None: db.close()

def init_db(): db = get_db() with app.open_resource('schema.sql', mode='r') as f: db.executescript(f.read()) db.commit()

Initialize DB if not exists

if not os.path.exists(DATABASE): with app.app_context(): init_db()

---------- Routes ----------

@app.route('/') def index(): return render_template('index.html')

--- Auth (very basic) ---

@app.route('/api/signup', methods=['POST']) def signup(): data = request.get_json() or {} username = data.get('username') password = data.get('password') if not username or not password: return jsonify({'error': 'username and password required'}), 400 db = get_db() cur = db.execute('SELECT id FROM users WHERE username=?', (username,)) if cur.fetchone(): return jsonify({'error': 'username exists'}), 400 pw_hash = generate_password_hash(password) db.execute('INSERT INTO users (username, password_hash) VALUES (?, ?)', (username, pw_hash)) db.commit() return jsonify({'status': 'ok'})

@app.route('/api/login', methods=['POST']) def login(): data = request.get_json() or {} username = data.get('username') password = data.get('password') if not username or not password: return jsonify({'error': 'username and password required'}), 400 db = get_db() cur = db.execute('SELECT id, password_hash FROM users WHERE username=?', (username,)) row = cur.fetchone() if not row or not check_password_hash(row['password_hash'], password): return jsonify({'error': 'invalid credentials'}), 401 # In production return JWT or session cookie. Here we return simple user id. return jsonify({'status': 'ok', 'user_id': row['id']})

--- Courses & Enrollment ---

@app.route('/api/courses', methods=['GET']) def list_courses(): db = get_db() cur = db.execute('SELECT id, title, description FROM courses') courses = [dict(row) for row in cur.fetchall()] return jsonify(courses)

@app.route('/api/enroll', methods=['POST']) def enroll(): data = request.get_json() or {} user_id = data.get('user_id') course_id = data.get('course_id') if not user_id or not course_id: return jsonify({'error': 'user_id and course_id required'}), 400 db = get_db() cur = db.execute('SELECT id FROM enrollments WHERE user_id=? AND course_id=?', (user_id, course_id)) if cur.fetchone(): return jsonify({'status': 'already enrolled'}) db.execute('INSERT INTO enrollments (user_id, course_id, progress) VALUES (?, ?, ?)', (user_id, course_id, 0)) db.commit() return jsonify({'status': 'enrolled'})

@app.route('/api/progress', methods=['POST']) def update_progress(): data = request.get_json() or {} user_id = data.get('user_id') course_id = data.get('course_id') progress = data.get('progress') if None in (user_id, course_id, progress): return jsonify({'error': 'user_id, course_id, progress required'}), 400 db = get_db() db.execute('UPDATE enrollments SET progress=? WHERE user_id=? AND course_id=?', (progress, user_id, course_id)) db.commit() return jsonify({'status': 'ok'})

@app.route('/api/mycourses', methods=['GET']) def mycourses(): user_id = request.args.get('user_id') if not user_id: return jsonify({'error': 'user_id required'}), 400 db = get_db() cur = db.execute(''' SELECT c.id, c.title, c.description, e.progress FROM courses c JOIN enrollments e ON c.id = e.course_id WHERE e.user_id = ? ''', (user_id,)) items = [dict(row) for row in cur.fetchall()] return jsonify(items)

if name == 'main': app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 5000)), debug=True)

---------- End of app.py ----------

FILE: schema.sql

""" -- SQLite schema for the example app DROP TABLE IF EXISTS users; DROP TABLE IF EXISTS courses; DROP TABLE IF EXISTS enrollments;

CREATE TABLE users ( id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE NOT NULL, password_hash TEXT NOT NULL );

CREATE TABLE courses ( id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT NOT NULL, description TEXT );

CREATE TABLE enrollments ( id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER NOT NULL, course_id INTEGER NOT NULL, progress INTEGER DEFAULT 0, FOREIGN KEY(user_id) REFERENCES users(id), FOREIGN KEY(course_id) REFERENCES courses(id) );

-- Seed sample courses INSERT INTO courses (title, description) VALUES ('Intro to Python', 'Basics of Python programming.'); INSERT INTO courses (title, description) VALUES ('Learning How to Learn', 'Techniques for accelerated learning.'); INSERT INTO courses (title, description) VALUES ('Cloud Basics', 'Intro to cloud concepts and deployment.'); """

FILE: requirements.txt

""" Flask==2.2.5 werkzeug==2.2.3

"""

FILE: Dockerfile

"""

Use official Python image

FROM python:3.11-slim WORKDIR /app COPY requirements.txt ./ RUN pip install --no-cache-dir -r requirements.txt COPY . /app ENV PORT=8080 EXPOSE 8080 CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app", "--workers", "2"] """

FILE: templates/index.html

""" <!doctype html>

<html>
  <head>
    <meta charset="utf-8" />
    <title>Lifelong Learning App - Demo</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <style>
      body { font-family: Arial, sans-serif; max-width: 800px; margin: 2rem auto; padding: 1rem; }
      input, button { padding: .5rem; margin: .25rem 0; }
      .course { border: 1px solid #ddd; padding: .5rem; margin: .5rem 0 }
    </style>
  </head>
  <body>
    <h1>Lifelong Learning — Demo</h1>
    <section id="auth">
      <h2>Sign up / Login</h2>
      <input id="username" placeholder="username" /> <br />
      <input id="password" placeholder="password" type="password" /> <br />
      <button onclick="signup()">Sign up</button>
      <button onclick="login()">Login</button>
      <div id="auth-result"></div>
    </section><section id="courses">
  <h2>Courses</h2>
  <div id="courses-list">Loading…</div>
</section>

<section id="mycourses">
  <h2>My Courses</h2>
  <div id="mycourses-list">Not signed in</div>
</section>

<script>
  let currentUserId = null;

  async function fetchCourses(){
    const res = await fetch('/api/courses');
    const data = await res.json();
    const el = document.getElementById('courses-list');
    el.innerHTML = data.map(c => `\n          <div class="course">\n            <strong>${c.title}</strong>\n            <p>${c.description}</p>\n            <button onclick="enroll(${c.id})">Enroll</button>\n          </div>\n        `).join('');
  }

  async function signup(){
    const username = document.getElementById('username').value;
    const password = document.getElementById('password').value;
    const res = await fetch('/api/signup', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username, password})});
    const data = await res.json();
    document.getElementById('auth-result').innerText = JSON.stringify(data);
  }

  async function login(){
    const username = document.getElementById('username').value;
    const password = document.getElementById('password').value;
    const res = await fetch('/api/login', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({username, password})});
    const data = await res.json();
    if(data.user_id){
      currentUserId = data.user_id;
      document.getElementById('auth-result').innerText = 'Logged in as user_id=' + currentUserId;
      loadMyCourses();
    } else {
      document.getElementById('auth-result').innerText = JSON.stringify(data);
    }
  }

  async function enroll(courseId){
    if(!currentUserId){ alert('Please login first'); return; }
    const res = await fetch('/api/enroll', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({user_id: currentUserId, course_id: courseId})});
    const data = await res.json();
    alert(JSON.stringify(data));
    loadMyCourses();
  }

  async function loadMyCourses(){
    if(!currentUserId) return;
    const res = await fetch('/api/mycourses?user_id=' + currentUserId);
    const data = await res.json();
    const el = document.getElementById('mycourses-list');
    if(data.error){ el.innerText = data.error; return; }
    el.innerHTML = data.map(c => `\n          <div class="course">\n            <strong>${c.title}</strong> — Progress: ${c.progress}%\n            <div>\n              <input type="range" min="0" max="100" value="${c.progress}" onchange="updateProgress(${c.id}, this.value)">\n            </div>\n          </div>\n        `).join('');
  }

  async function updateProgress(courseId, progress){
    if(!currentUserId) return;
    await fetch('/api/progress', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({user_id: currentUserId, course_id: courseId, progress: Number(progress)})});
    loadMyCourses();
  }

  // Initialize
  fetchCourses();
</script>

  </body>
</html>
"""FILE: README.md

"""

Lifelong Learning — Minimal Cloud App (Flask)

This is a beginner-friendly example of a cloud-deployable web app for learning projects.

Run locally

1. python -m venv venv


2. source venv/bin/activate


3. pip install -r requirements.txt


4. flask run



Build & Deploy using Docker

1. docker build -t lifelong-app .


2. docker run -p 8080:8080 lifelong-app



Then open http://localhost:8080

Next steps / Improvements

Replace simple auth with JWT and secure password reset flows

Move to cloud DB (Cloud SQL, Firestore) for scaling

Add CI/CD (GitHub Actions) to auto-deploy

Add tests and input validation

Add analytics (Google Analytics / Firebase Analytics) """


