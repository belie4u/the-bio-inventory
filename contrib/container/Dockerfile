# Multi-stage: Builder for deps and frontend
FROM python:3.11-slim-trixie@sha256:1d6131b5d479888b43200645e03a78443c7157efbdb730e6b48129740727c312 AS builder

ARG commit_tag=""
ARG commit_hash=""
ARG commit_date=""
ARG data_dir="data"

ENV PYTHONUNBUFFERED=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV INVOKE_RUN_SHELL="/bin/bash"
ENV INVENTREE_DOCKER="true"

ENV INVENTREE_HOME="/home/inventree"
ENV INVENTREE_DATA_DIR="${INVENTREE_HOME}/${data_dir}"
ENV INVENTREE_BACKEND_DIR="${INVENTREE_HOME}/src/backend"
ENV INVENTREE_FRONTEND_DIR="${INVENTREE_HOME}/src/frontend"

# Install system deps in one layer
RUN apt-get update && apt-get install -y --no-install-recommends \
    git gettext libldap2 wget curl ssh \
    weasyprint libpango-1.0-0 libcairo2 poppler-utils \
    postgresql-client mariadb-client \
    fontconfig fonts-freefont-ttf fonts-terminus fonts-noto-core fonts-noto-cjk \
    pkg-config build-essential \
    libldap2-dev libsasl2-dev libssl-dev \
    libmariadb-dev \
    nodejs npm \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/lib/python3/dist-packages/numpy \
    && rm -rf /usr/lib/python3/dist-packages/scipy \
    && rm -rf /usr/lib/python3/dist-packages/sympy

# Install invoke early
RUN pip install --no-cache-dir -U invoke

WORKDIR ${INVENTREE_HOME}

# Copy requirements FIRST for caching
COPY contrib/container/requirements.txt base_requirements.txt
COPY src/backend/requirements.txt ${INVENTREE_BACKEND_DIR}/requirements.txt

# Install Python deps (without --require-hashes for speed; add back if needed)
RUN pip install --no-cache-dir -r base_requirements.txt && \
    pip install --no-cache-dir -r ${INVENTREE_BACKEND_DIR}/requirements.txt && \
    pip cache purge

# Install Yarn/Node and compile frontend
RUN npm install -g n yarn --ignore-scripts && \
    yarn config set network-timeout 600000 -g && \
    n lts
COPY src ${INVENTREE_HOME}/src  
COPY tasks.py ./
RUN cd "${INVENTREE_HOME}" && invoke int.frontend-compile --extract

# Production stage: Copy only essentials
FROM python:3.11-slim-trixie@sha256:1d6131b5d479888b43200645e03a78443c7157efbdb730e6b48129740727c312 AS production

ARG commit_tag=""
ARG commit_hash=""
ARG commit_date=""
ARG data_dir="data"

ENV PYTHONUNBUFFERED=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV INVOKE_RUN_SHELL="/bin/bash"
ENV INVENTREE_DOCKER="true"
ENV INVENTREE_DEBUG=False

ENV INVENTREE_HOME="/home/inventree"
ENV INVENTREE_DATA_DIR="${INVENTREE_HOME}/${data_dir}"
ENV INVENTREE_STATIC_ROOT="${INVENTREE_DATA_DIR}/static"
ENV INVENTREE_MEDIA_ROOT="${INVENTREE_DATA_DIR}/media"
ENV INVENTREE_BACKUP_DIR="${INVENTREE_DATA_DIR}/backup"
ENV INVENTREE_PLUGIN_DIR="${INVENTREE_DATA_DIR}/plugins"
ENV INVENTREE_BACKEND_DIR="${INVENTREE_HOME}/src/backend"

ENV INVENTREE_CONFIG_FILE="${INVENTREE_DATA_DIR}/config.yaml"
ENV INVENTREE_SECRET_KEY_FILE="${INVENTREE_DATA_DIR}/secret_key.txt"
ENV INVENTREE_OIDC_PRIVATE_KEY_FILE="${INVENTREE_DATA_DIR}/oidc.pem"
ENV INVENTREE_PLUGIN_FILE="${INVENTREE_DATA_DIR}/plugins.txt"

ENV INVENTREE_GUNICORN_WORKERS="4"
ENV INVENTREE_BACKGROUND_WORKERS="4"
ENV INVENTREE_WEB_ADDR="0.0.0.0"
ENV INVENTREE_WEB_PORT="8000"

LABEL org.opencontainers.image.vendor="inventree" \
      org.opencontainers.image.title="InvenTree backend server" \
      org.opencontainers.image.description="InvenTree is the open-source inventory management system" \
      org.opencontainers.image.url="https://inventree.org" \
      org.opencontainers.image.documentation="https://docs.inventree.org" \
      org.opencontainers.image.source="https://github.com/inventree/InvenTree" \
      org.opencontainers.image.revision=${commit_hash} \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.version=${commit_tag}

# Install minimal runtime system deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    git gettext libldap2 wget curl ssh \
    weasyprint libpango-1.0-0 libcairo2 poppler-utils \
    postgresql-client mariadb-client \
    fontconfig fonts-freefont-ttf fonts-terminus fonts-noto-core fonts-noto-cjk \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p ${INVENTREE_HOME}
WORKDIR ${INVENTREE_HOME}

# Copy built artifacts from builder
COPY --from=builder /root/.cache/pip /root/.cache/pip 
COPY --from=builder ${INVENTREE_BACKEND_DIR}/InvenTree ${INVENTREE_BACKEND_DIR}/InvenTree
COPY --from=builder ${INVENTREE_BACKEND_DIR}/InvenTree/web/static/web ${INVENTREE_BACKEND_DIR}/InvenTree/web/static/web
COPY contrib/container/gunicorn.conf.py .
COPY contrib/container/init.sh .
RUN chmod +x init.sh

ENV INVENTREE_COMMIT_HASH="${commit_hash}"
ENV INVENTREE_COMMIT_DATE="${commit_date}"

EXPOSE 8000

ENTRYPOINT ["/bin/bash", "./init.sh"]
CMD ["gunicorn", "--chdir", "${INVENTREE_BACKEND_DIR}/InvenTree", "-c", "${INVENTREE_HOME}/gunicorn.conf.py", "InvenTree.wsgi", "-b", "0.0.0.0:${INVENTREE_WEB_PORT}"]