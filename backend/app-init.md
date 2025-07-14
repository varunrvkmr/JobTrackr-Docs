---
### `app/__init__.py` – Application Factory Overview

This module defines the Flask application factory function and serves as the entry point for initializing and configuring the backend of the system.

---

#### Imports

The file imports essential modules and extensions, including:

* Core Flask modules: `Flask`, `jsonify`
* Environment management: `load_dotenv`, `os`, `json`
* Configuration: `Config` from `config.py`
* Extensions:

  * `CORS` for Cross-Origin Resource Sharing
  * `Migrate` for database migrations
  * `JWTManager` for authentication
  * `db` and `oauth` from `app.extensions`
* Route blueprints from `app.routes.*`

<details>
<summary>Show import block</summary>

```python
from flask import Flask, jsonify
from flask_cors import CORS
from flask_migrate import Migrate
from flask_jwt_extended import JWTManager
from config import Config
from app.extensions import db, oauth
from app.routes.jobs_routes import job_bp
from app.routes.letter_generator import letter_generator_bp
from app.routes.user_profiles_routes import user_profiles_routes_bp
from app.routes.auth_routes import auth_bp
from app.routes.proxy_routes import proxy_bp
from app.routes.embed_routes import embed_bp
from app.routes.autofill_routes import autofill_bp
from app.routes.chatbot_routes import chatbot_bp
from dotenv import load_dotenv
import os
import json
```

</details>

---

#### `create_app()` Function

The `create_app` function is the application factory that initializes and returns a configured Flask app instance.

---

##### Environment and Configuration

Loads configuration from a combination of the `Config` object and `.env` file:

<details>
<summary>Show configuration block</summary>

```python
load_dotenv()
app = Flask(__name__)
app.config.from_object(Config)
app.config["SECRET_KEY"] = os.getenv("SECRET_KEY", "fallback-secret2")
app.config["JWT_SECRET_KEY"] = os.getenv("JWT_SECRET_KEY", "fallback-secret")
app.config["SQLALCHEMY_DATABASE_URI"] = os.getenv("DATABASE_URL")
app.config["OPENAI_API_KEY"] = os.getenv("OPENAI_API_KEY")
```

</details>

---

##### JWT Cookie Configuration

Configures JWT for cookie-based authentication:

<details>
<summary>Show JWT config</summary>

```python
app.config["JWT_TOKEN_LOCATION"] = ["cookies"]
app.config["JWT_COOKIE_SECURE"] = False
app.config["JWT_COOKIE_SAMESITE"] = "Lax"
app.config["JWT_COOKIE_CSRF_PROTECT"] = False
```

</details>

---

#### CORS Configuration

Defines cross-origin permissions for frontend and Chrome extension access:

<details>
<summary>Show CORS config</summary>

```python
CORS(app, resources={
    r"/api/*": {
        "origins": [
            "chrome-extension://micccolehgefkhdjlalncmomknlankaj",
            "http://localhost:3000"
        ],
        "supports_credentials": True
    },
    r"/proxy/*": {
        "origins": "*",
        "supports_credentials": False
    }
})
```

</details>

---

#### Extension Initialization

Initializes Flask extensions for database, authentication, and OAuth:

<details>
<summary>Show extension init block</summary>

```python
db.init_app(app)
jwt.init_app(app)
oauth.init_app(app)
Migrate(app, db)
```

</details>

---

#### Database Table Creation

Creates all database tables in development mode:

<details>
<summary>Show table creation block</summary>

```python
with app.app_context():
    db.create_all()
    print("Database tables created successfully")
```

</details>

---

#### Threshold Configuration

Loads a custom JSON file for field classification thresholds:

<details>
<summary>Show thresholds config block</summary>

```python
thresholds_path = os.path.join(app.root_path, 'config', 'thresholds.json')
with open(thresholds_path) as f:
    app.config['FIELD_TYPE_THRESHOLDS'] = json.load(f)
```

</details>

---

#### Blueprint Registration

Registers route blueprints for modular organization:

<details>
<summary>Show blueprint registration block</summary>

```python
app.register_blueprint(job_bp, url_prefix="/api/jobs")
app.register_blueprint(letter_generator_bp, url_prefix="/api/letter-generator")
app.register_blueprint(user_profiles_routes_bp, url_prefix="/api/user-profile")
app.register_blueprint(auth_bp, url_prefix="/api/auth")
app.register_blueprint(proxy_bp, url_prefix="/proxy")
app.register_blueprint(embed_bp, url_prefix="/api/embed")
app.register_blueprint(autofill_bp, url_prefix="/api/autofill")
app.register_blueprint(chatbot_bp, url_prefix="/api/chat")
```

</details>

---

#### Health Check Route

Defines a simple route to check backend availability:

<details>
<summary>Show health check route</summary>

```python
@app.route('/api/', methods=['GET'])
def backend_status():
    return jsonify({"message": "✅ Backend is running on port 5050!"})
```

</details>

---
