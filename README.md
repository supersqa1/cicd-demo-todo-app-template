# CI/CD Demo To-Do App

A small Flask application used as the starting point for the SuperSQA CI/CD tutorial series.
Start here with the Dockerized application, then build the CI/CD pipeline by following the tutorial. This repository does not include a completed pipeline.

## Run locally

Use Python 3.12 or newer and run these commands from the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python app.py
```

On Windows, create the environment with `py -3.12 -m venv .venv` and activate it with `.venv\Scripts\Activate.ps1` in PowerShell.

Open http://localhost:5151. Stop the server with Ctrl+C.

## Run with Docker

Install Docker with Compose (Docker Desktop includes both), then run:

```bash
docker compose up --build -d
```

Open http://localhost:5151. This binds to your local machine only.

```bash
docker compose logs -f
docker compose down
```

Tasks live in a named Docker volume mounted at `/app/tasks`, so they survive container replacement and `docker compose down`. `docker compose down --volumes` deletes the task data.

The same Dockerfile builds natively on ARM64 (Apple Silicon) and AMD64 (Intel/AMD). No platform override is needed for local use.

### Build for either architecture

With Docker Desktop, or a Buildx builder configured with both architectures:

```bash
docker buildx build --platform linux/arm64 -t cicd-demo-todo-app:arm64 --load .
docker buildx build --platform linux/amd64 -t cicd-demo-todo-app:amd64 --load .
```

To publish both architectures under one image tag later, replace `YOUR_NAMESPACE` with your registry namespace and log in to that registry first:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/YOUR_NAMESPACE/cicd-demo-todo-app:latest --push .
```

Docker selects the matching architecture when pulling that tag. See [Docker's multi-platform build documentation](https://docs.docker.com/build/building/multi-platform/) for builder setup on Linux.

## What the app does

- Create tasks with a title and description.
- Change their status to To-Do, In Progress, or Completed.
- Delete tasks.

The app starts empty. It creates a local `tasks/` directory when you add your first task. Generated task files are ignored by Git.

## Starting-point scope

This preserves the original application behavior and templates. Docker runs the app with Gunicorn as a non-root user. Automated tests and CI/CD configurations will be added in later changes.

This is a local teaching demo, not yet ready for public hosting: it has no authentication, and the non-Docker startup uses Flask's debug server. Task titles are used as filenames without validation, duplicate titles overwrite tasks, and multiline descriptions do not round-trip correctly. Storage and deployment improvements are planned before hosting it.

## Source

Copied from [supersqa1/todo-app-flask-demo](https://github.com/supersqa1/todo-app-flask-demo), commit `1e918f19c0a42222cdb47687a71f700971f69619`.
Only the active application, templates, dependency list, and ignore rules were carried over. The original Git history, sample task data, and unused `views/views.py` were omitted.
