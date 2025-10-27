````{"id":"52741","variant":"standard","title":"README - Flask Bootstrap Docker Explanation"}
# Flask + Bootstrap + Docker — Project README & Troubleshooting

**Author:** Cephas Okuku  
**Purpose:** Minimal Flask app using Bootstrap, wrapped in Docker. This README explains the project structure, why each file exists, how to run it, and how to fix common mistakes (ports, missing packages, rebuilds). Keep this for future reference.

---

## Project structure

```
flask-bootstrap-docker/
├── app.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .gitignore
├── static/
│   ├── css/
│   │   └── bootstrap.min.css
│   └── js/
│       └── bootstrap.bundle.min.js
└── templates/
    └── index.html
```

---

## Files & their purpose

### `app.py`
Main Flask app. The `host="0.0.0.0"` makes Flask listen on all container network interfaces (necessary for Docker).

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html")

if __name__ == "__main__":
    # Use an explicit port (e.g., 8080) so you avoid common reserved ports on Windows.
    app.run(host="0.0.0.0", port=8080, debug=True)
```

**Why port 8080 above?**  
On Windows, port 5000 may be reserved or blocked. Using 8080 is commonly safe. You can change it to any free port (see Ports section).

---

### `requirements.txt`
Specifies Python dependencies to install inside the container — this ensures the container environment is reproducible.

```
Flask==3.0.3
```

**Why it's important:**  
Docker layers cache installs; if you change dependencies but don't rebuild properly, container will still lack the new packages. Always include `requirements.txt` in the image build so the image has all needed Python packages.

---

### `Dockerfile`
Builds the container image and installs dependencies.

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Copy and install Python dependencies first (layer caching benefit)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application source
COPY . .

# Expose chosen port (for documentation; mapping happens in 'docker run' or compose)
EXPOSE 8080

CMD ["python", "app.py"]
```

**Notes**
- COPY `requirements.txt` before copying all files so Docker caches the pip install layer. If `requirements.txt` doesn't change, rebuilds are faster.
- `EXPOSE` is documentation only — actual host-to-container port mapping is done with `-p` or in Compose `ports:`.

---

### `docker-compose.yml`
Simple compose file that builds the image and maps host port `8080` to container port `8080`.

```yaml
services:
  web:
    build: .
    container_name: flask_bootstrap_app
    ports:
      - "8080:8080"
    volumes:
      - .:/app
    environment:
      - FLASK_ENV=development
```

**Important note about `version`:**  
Newer Docker Compose (the `docker` CLI v2 plugin) warns that the top-level `version` field is obsolete. You can remove `version: "3.9"` if you used it before. Compose will still work without it.

---

### `templates/index.html`
Uses Bootstrap files stored under `static/`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Flask Bootstrap Demo</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/bootstrap.min.css') }}">
</head>
<body>
  <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container-fluid">
      <a class="navbar-brand" href="#">FlaskApp</a>
    </div>
  </nav>

  <div class="container mt-5">
    <div class="text-center">
      <h1 class="display-4">Welcome to Flask + Bootstrap</h1>
      <p class="lead">This is a simple Flask app served in Docker.</p>
    </div>
  </div>

  <script src="{{ url_for('static', filename='js/bootstrap.bundle.min.js') }}"></script>
</body>
</html>
```

---

### `.dockerignore` (recommended)
Prevents unnecessary files from being copied into the image.

```
__pycache__
*.pyc
.env
venv/
.git
```

---

### `.gitignore` (recommended)
Hide local/OS/dev files from git.

```
__pycache__/
*.pyc
venv/
.env
.DS_Store
```

---

## Ports — what they are & why they matter

- **Container port** is the port your app listens on inside the container (here `8080`).
- **Host port** is the port on your machine (your Windows host) that forwards traffic into the container.
- `-p HOST:CONTAINER` or `ports: - "HOST:CONTAINER"` in Compose maps them.

Example: `-p 8080:8080` maps `http://localhost:8080` (host) to port `8080` inside the container.

**Why you saw the binding error earlier**  
`bind: An attempt was made to access a socket in a way forbidden by its access permissions` means the host refused to bind to that host port. Causes include:
- Another process is already listening on that host port.
- OS reserved the port (Windows port reservation).
- Insufficient permissions or firewall/security software blocked the bind.

**How to check and fix:**
- See which process uses the port (Windows CMD or PowerShell):
  ```
  netstat -ano | findstr :5000
  ```
  Then `taskkill /PID <pid> /F` (if safe).
- Or change host port mapping: `docker run -d -p 5050:8080 image` then open `http://localhost:5050`.

**When you can skip mapping ports:**
- If the container only communicates with other containers (internal-only microservice), you can omit host port mapping and use Docker networks instead. For local browser testing you must map a host port.

---

## Why `ModuleNotFoundError: No module named 'flask'` happened

- The container image didn’t have Flask installed. This happens when:
  - `requirements.txt` was missing or not copied in the image build step.
  - You changed `requirements.txt` but didn’t rebuild the image.
  - You used a different Python base image or virtual env issues inside the build.

**Fix:**
1. Ensure `requirements.txt` exists in repo root with `Flask==...`.
2. Rebuild the image so pip runs during build:
   - With docker compose: `docker compose up --build`
   - With docker: `docker build -t flask-bootstrap-app . && docker run -p 8080:8080 flask-bootstrap-app`

---

## Rebuild & caching notes (important)
Docker caches layers. If you:
- Change `app.py` only — with `COPY . .` last and `requirements.txt` copied earlier, image rebuild is fast and pip will not re-run.
- Change `requirements.txt` — Docker must reinstall packages. Use:
  ```
  docker compose up --build
  ```
  to force a rebuild. Or `docker compose build --no-cache` to start fresh.

---

## Useful commands

Build and run with compose (foreground logs):
```bash
docker compose up --build
```

Run detached:
```bash
docker compose up -d --build
```

Stop and remove:
```bash
docker compose down
```

Build with docker directly:
```bash
docker build -t flask-bootstrap-app .
docker run -d -p 8080:8080 --name flask_bootstrap_app flask-bootstrap-app
```

Check logs:
```bash
docker logs -f flask_bootstrap_app
```

Check running containers:
```bash
docker ps
```

Inspect a running container shell:
```bash
docker exec -it flask_bootstrap_app sh
```

Check port usage on Windows:
```powershell
netstat -ano | findstr :8080
```

Kill a process by PID (Windows CMD):
```cmd
taskkill /PID 12345 /F
```

---

## Git: quick copy-to-GitHub steps

```bash
git init
git add .
git commit -m "Initial Flask bootstrap docker app"
git branch -M main
git remote add origin git@github.com:yourusername/flask-bootstrap-docker.git
git push -u origin main
```

If using HTTPS remote:
```bash
git remote add origin https://github.com/yourusername/flask-bootstrap-docker.git
git push -u origin main
```

---

## FAQ / Troubleshooting checklist

1. **App returns `ModuleNotFoundError: No module named 'flask'`**
   - Ensure `requirements.txt` exists. Rebuild image: `docker compose up --build`.

2. **`ports are not available` / bind error**
   - Change host port (e.g., `5050:8080`), or free the port using `netstat` + `taskkill`, or check Windows excluded port ranges: `netsh interface ipv4 show excludedportrange protocol=tcp`.

3. **`docker compose` warns about `version`**
   - Remove the `version:` top-level key from `docker-compose.yml`. Compose will work without it.

4. **I modify code but changes don't show**
   - If you're using volumes (`- .:/app`) in compose it should reflect changes immediately (host bind). If you don't have the volume, you need to rebuild the image to see code changes.

---

## Final tips & good practices

- Keep `requirements.txt` up to date and deterministic (pin versions).
- Use `EXPOSE` for documentation, and always map host → container ports during local development.
- Use a `.dockerignore` to keep build context small (faster builds).
- Use environment variables for ports/other config (example below) to avoid hardcoding.

**Example: use env var for port**
`docker-compose.yml` (snippet)
```yaml
services:
  web:
    environment:
      - PORT=8080
    ports:
      - "${HOST_PORT:-8080}:${PORT:-8080}"
```
`app.py` (snippet)
```python
import os
port = int(os.getenv("PORT", 8080))
app.run(host="0.0.0.0", port=port)
```

---

Which would you like next?  
````
