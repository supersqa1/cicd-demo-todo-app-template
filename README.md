# CI/CD Demo To-Do App

A small Dockerized Flask app for the SuperSQA CI/CD tutorial series. Create, view, update, and delete tasks while learning how to build a deployment pipeline. This is the starter application; no CI/CD pipeline is included yet.

## Start your own repository

1. Click **Use this template → Create a new repository** on GitHub.
2. Choose your repository name and owner.
3. Clone **your new repository** and open its directory in a terminal.

## Project structure

```text
app/
  main.py             # Flask routes and file-based task storage
  templates/          # HTML pages rendered by Flask
  requirements.txt    # Python dependencies
  __init__.py         # Python package marker
Dockerfile            # Application image
compose.yaml          # Local container and persistent task volume
.dockerignore         # Files excluded from the image build
.gitignore            # Files excluded from Git
README.md
```

`app/templates/` contains the app's HTML pages. It is unrelated to GitHub's repository template feature.

## Run with Docker (recommended)

Install Docker with Compose (included in Docker Desktop). From the repository root:

```bash
docker compose up --build -d
```

Open http://localhost:5151. The app is accessible only from your local machine.

```bash
docker compose logs -f   # View logs; Ctrl+C stops following logs
docker compose down     # Stop and remove containers; keep task data
```

Tasks are saved as text files, not in a database. Docker stores them in a named volume mounted at `/app/tasks`, so they survive container replacement. **`docker compose down --volumes` deletes that data.**

## Run without Docker

Use Python 3.12 or newer. From the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r app/requirements.txt
python -m app.main
```

On Windows, use `py -3.12 -m venv .venv` and activate with `.venv\Scripts\Activate.ps1` in PowerShell. Open http://localhost:5151; stop with Ctrl+C. Local task files are created in `tasks/` and ignored by Git. They are separate from Docker's volume.

## ARM64 and AMD64 builds

Docker Compose builds for your machine automatically, including Apple Silicon and Intel/AMD. To explicitly build either platform with a compatible Buildx builder:

```bash
docker buildx build --platform linux/arm64 -t cicd-demo-todo-app:arm64 --load .
docker buildx build --platform linux/amd64 -t cicd-demo-todo-app:amd64 --load .
```

## Demo scope

Docker runs Gunicorn as a non-root user. The direct Python command runs Flask's development server. This starter has no authentication and is not ready for public hosting: task titles become filenames without validation, duplicate titles overwrite tasks, and multiline descriptions are not handled correctly. These limitations will be addressed before public deployment.

## Source

Adapted from [supersqa1/todo-app-flask-demo](https://github.com/supersqa1/todo-app-flask-demo), commit `1e918f19c0a42222cdb47687a71f700971f69619`. Sample data and unused duplicate code were omitted.
