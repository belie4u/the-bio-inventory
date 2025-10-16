# ===============================
# STAGE 1 — Builder
# ===============================
FROM python:3.11-slim-trixie AS builder

ENV PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# 🧩 Add pkg-config here (critical fix)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc pkg-config \
    libldap2-dev libsasl2-dev libssl-dev \
    libmariadb-dev libpq-dev libffi-dev zlib1g-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /install

# Copy requirements first (for caching)
COPY contrib/container/requirements.txt base_requirements.txt
COPY src/backend/requirements.txt backend_requirements.txt

# Install Python deps (optimized for prebuilt wheels)
RUN pip install --upgrade pip setuptools wheel \
 && pip install --prefer-binary -r base_requirements.txt \
 && pip install --prefer-binary -r backend_requirements.txt

# ===============================
# STAGE 2 — Production
# ===============================
FROM python:3.11-slim-trixie AS production

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

# Install only what’s needed at runtime
RUN apt-get update && apt-get install -y --no-install-recommends \
    git gettext wget curl \
    postgresql-client mariadb-client \
    weasyprint libpango-1.0-0 libcairo2 poppler-utils \
    fontconfig fonts-noto-core \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy Python packages from builder
COPY --from=builder /usr/local /usr/local

WORKDIR ${INVENTREE_HOME}

# Copy your backend code
COPY src/backend/InvenTree ${INVENTREE_BACKEND_DIR}/InvenTree
COPY tasks.py contrib/container/gunicorn.conf.py contrib/container/init.sh ./
RUN chmod +x init.sh

EXPOSE ${INVENTREE_WEB_PORT}

HEALTHCHECK CMD curl -f http://localhost:${INVENTREE_WEB_PORT}/api/ || exit 1

ENTRYPOINT ["/bin/bash", "./init.sh"]
CMD ["gunicorn", "--chdir", "/inventree/src/backend/InvenTree", "-c", "/inventree/gunicorn.conf.py", "InvenTree.wsgi", "-b", "0.0.0.0:8000"]
