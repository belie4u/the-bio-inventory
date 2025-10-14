# Web process: gunicorn
web: gunicorn --chdir $APP_HOME/src/backend/InvenTree -c src/backend/InvenTree/gunicorn.conf.py InvenTree.wsgi -b 0.0.0.0:$PORT

# Worker process: qcluster
worker: python src/backend/InvenTree/manage.py qcluster

# Invoke commands
invoke: invoke

# CLI: Provided for backwards compatibility
cli: invoke