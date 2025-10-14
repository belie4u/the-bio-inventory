# Production stage
FROM python:3.11-slim-bookworm AS production

# Build arguments
ARG commit_tag=""
ARG commit_hash=""
ARG commit_date=""
ARG data_dir="data"

# Environment variables
ENV PYTHONUNBUFFERED=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV INVOKE_RUN_SHELL="/bin/bash"
ENV INVENTREE_DOCKER="true"
ENV INVENTREE_DEBUG=False

# InvenTree paths
ENV INVENTREE_HOME="/home/inventree"
ENV INVENTREE_DATA_DIR="${INVENTREE_HOME}/${data_dir}"
ENV INVENTREE_STATIC_ROOT="${INVENTREE_DATA_DIR}/static"
ENV INVENTREE_MEDIA_ROOT="${INVENTREE_DATA_DIR}/media"
ENV INVENTREE_BACKUP_DIR="${INVENTREE_DATA_DIR}/backup"
ENV INVENTREE_PLUGIN_DIR="${INVENTREE_DATA_DIR}/plugins"
ENV INVENTREE_BACKEND_DIR="${INVENTREE_HOME}/src/backend"

# InvenTree configuration files
ENV INVENTREE_CONFIG_FILE="${INVENTREE_DATA_DIR}/config.yaml"
ENV INVENTREE_SECRET_KEY_FILE="${INVENTREE_DATA_DIR}/secret_key.txt"
ENV INVENTREE_OIDC_PRIVATE_KEY_FILE="${INVENTREE_DATA_DIR}/oidc.pem"
ENV INVENTREE_PLUGIN_FILE="${INVENTREE_DATA_DIR}/plugins.txt"

# Worker configuration
ENV INVENTREE_GUNICORN_WORKERS="4"
ENV INVENTREE_BACKGROUND_WORKERS="4"

# Default web server address:port
ENV INVENTREE_WEB_ADDR="0.0.0.0"
ENV INVENTREE_WEB_PORT="8000"

# Labels
LABEL org.opencontainers.image.vendor="inventree" \
      org.opencontainers.image.title="InvenTree backend server" \
      org.opencontainers.image.description="InvenTree is the open-source inventory management system" \
      org.opencontainers.image.url="https://inventree.org" \
      org.opencontainers.image.documentation="https://docs.inventree.org" \
      org.opencontainers.image.source="https://github.com/inventree/InvenTree" \
      org.opencontainers.image.revision=${commit_hash} \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.version=${commit_tag}

# Install minimal runtime system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    weasyprint libpango-1.0-0 libcairo2 poppler-utils \
    postgresql-client \
    fontconfig fonts-freefont-ttf fonts-terminus fonts-noto-core fonts-noto-cjk \
    libldap2-dev libsasl2-dev libssl-dev libpq-dev \
    gcc python3-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Create working directory
RUN mkdir -p ${INVENTREE_HOME}
WORKDIR ${INVENTREE_HOME}

# Copy requirements first for caching
COPY contrib/container/requirements.txt base_requirements.txt
COPY src/backend/requirements.txt ${INVENTREE_BACKEND_DIR}/requirements.txt

# Install Python dependencies
RUN pip install --no-cache-dir -r base_requirements.txt && \
    pip install --no-cache-dir -r ${INVENTREE_BACKEND_DIR}/requirements.txt && \
    pip cache purge

# Copy source and configuration files
COPY src/backend/InvenTree ${INVENTREE_BACKEND_DIR}/InvenTree
COPY src/backend/InvenTree/web/static/web ${INVENTREE_BACKEND_DIR}/InvenTree/web/static/web
COPY contrib/container/gunicorn.conf.py .
COPY contrib/container/init.sh .
COPY tasks.py .
RUN chmod +x init.sh

# Set commit information
ENV INVENTREE_COMMIT_HASH="${commit_hash}"
ENV INVENTREE_COMMIT_DATE="${commit_date}"

# Expose port
EXPOSE 8000

# Entrypoint
ENTRYPOINT ["/bin/bash", "./init.sh"]

# Launch production server
CMD ["gunicorn", "--chdir", "${INVENTREE_BACKEND_DIR}/InvenTree", "-c", "${INVENTREE_HOME}/gunicorn.conf.py", "InvenTree.wsgi", "-b", "0.0.0.0:${INVENTREE_WEB_PORT}"]