# ============================================================================
# Dockerfile — CloudBoard Flask Application
# ============================================================================
#
# This Dockerfile builds the Flask application image.
# It is referenced from compose.yaml under the "app" service with:
#
#   build:
#     context: ./app
#
# CONCEPTS FOR STUDENTS:
#   - FROM       : The base image we start from. We use a slim Python image
#                  to keep the final image small.
#   - WORKDIR    : Sets the working directory inside the container.
#                  All subsequent commands run from this path.
#   - COPY       : Copies files from the host into the image.
#                  We copy requirements.txt first so Docker can cache the
#                  pip install layer — it only re-runs when requirements change.
#   - RUN        : Executes a command during the build (not at runtime).
#   - EXPOSE     : Documents which port the app listens on.
#                  (Docker doesn't actually open the port — that's done in
#                   compose.yaml with "ports:")
#   - CMD        : The default command to run when the container starts.
#                  We use Gunicorn (a production WSGI server) instead of
#                  Flask's built-in dev server.
# ============================================================================

# ── Stage: base image ───────────────────────────────────────────────────────
# python:3.12-slim is a minimal Debian image with Python 3.12 pre-installed.
# "slim" means no build tools or documentation — keeps the image lean.
FROM python:3.12-slim

# ── Set working directory ───────────────────────────────────────────────────
# All files we COPY and all RUN commands will use /app as the base path.
WORKDIR /app

# ── Install Python dependencies ─────────────────────────────────────────────
# We copy requirements.txt BEFORE the rest of the source code.
# Docker builds images in layers; if requirements.txt hasn't changed,
# Docker uses the cached layer and skips the pip install — much faster!
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt
# --no-cache-dir keeps the image smaller by not saving pip's download cache.

# ── Copy application source code ─────────────────────────────────────────────
# The dot "." means "copy everything in the build context (./app on the host)
# into the current WORKDIR (/app in the image)".
COPY . .

# ── Document the listening port ──────────────────────────────────────────────
# This is metadata only — it doesn't publish the port.
# The actual port mapping is defined in compose.yaml.
EXPOSE 5000

# ── Start the application ────────────────────────────────────────────────────
# Gunicorn is a production WSGI server. It handles multiple concurrent
# requests better than Flask's built-in server.
#
#   --bind 0.0.0.0:5000   → listen on all network interfaces, port 5000
#   --workers 2           → 2 worker processes (enough for a workshop)
#   app:app               → the "app" object inside the "app.py" module
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
