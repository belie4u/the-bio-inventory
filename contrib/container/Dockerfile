# ===============================
# STAGE 1 — Builder
# ===============================
FROM python:3.11-slim-trixie AS builder

# Environment setup
ENV PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Install only what is needed to build Python deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libldap2-dev libsasl2-dev libssl-dev libmariadb-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /install

# Copy dependency files first (to enable Docker caching)
COPY contrib/container/requirements.txt base_requirements.txt
COPY src/backend/requirements.txt backend_requirements.txt

# Upgrade pip and install dependencies using prebuilt wheels if possible
RUN pip install --upgrade pip wheel setuptools \
 && pip install --prefer-binary -r base_requirements.txt \
 && pip install --prefer-binary -r backend_requirements.txt

# ===============================
# STAGE 2 — Production
# ===============================
FROM python:3.11-slim-trixie AS production

# Environment variables
ENV PYTHONUNBUFFERED=1 \
    INVENTREE_DOCKER=true \
    INVENTREE_DEBUG=False \
    INVENTREE_HOME="/inventree" \
    INVENTREE_WEB_PORT=8000 \
    INVENTREE_WEB_ADDR="0.0.0.0" \
    INVENTREE_DATA_DIR="/inventree/data" \
    INVENTREE_BACKEND_DIR="/inventree/src/backend" \
    INVENTREE_STATIC_ROOT="/inventree/data/static" \
    INVENTREE_MEDIA_ROOT="/inventree/data/media" \
    INVENTREE_BACKUP_DIR="/inventree/data/backup" \
    INVENTREE_PLUGIN_DIR="/inventree/data/plugins"

# Install lightweight runtime system dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    git gettext wget curl \
    postgresql-client mariadb-client \
    weasyprint libpango-1.0-0 libcairo2 poppler-utils \
    fontconfig fonts-noto-core \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy installed Python packages from builder stage
COPY --from=builder /usr/local /usr/local

# Create working directory
WORKDIR ${INVENTREE_HOME}

# Copy application source
COPY src/backend/InvenTree ${INVENTREE_BACKEND_DIR}/InvenTree
COPY tasks.py contrib/container/gunicorn.conf.py contrib/container/init.sh ./
RUN chmod +x init.sh

# Expose port for Railway
EXPOSE ${INVENTREE_WEB_PORT}

# Healthcheck (optional)
HEALTHCHECK CMD curl -f http://localhost:${INVENTREE_WEB_PORT}/api/ || exit 1

# Entrypoint and default command
ENTRYPOINT ["/bin/bash", "./init.sh"]
CMD ["gunicorn", "--chdir", "/inventree/src/backend/InvenTree", "-c", "/inventree/gunicorn.conf.py", "InvenTree.wsgi", "-b", "0.0.0.0:8000"]
