FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Instalar Python 3.12, git, jq, Node.js y npm desde los repositorios oficiales de Ubuntu
RUN apt-get update && \
    apt-get install -y python3 python3-venv python3-pip libssl-dev git jq curl ca-certificates nodejs npm && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Clonar el repositorio para obtener app, assets y scripts
RUN git clone https://github.com /tmp/repo && \
    cp -r /tmp/repo/app /app/src && \
    cp -r /tmp/repo/app /app/app && \
    cp -r /tmp/repo/assets /app/assets && \
    cp -r /tmp/repo/scripts /app/scripts && \
    rm -rf /tmp/repo && \
    chmod +x /app/scripts/ask_create_skill.sh || true

# Instalar dependencias de Python en entorno virtual
RUN python3 -m venv venv && \
    . venv/bin/activate && \
    pip install --upgrade pip && \
    pip install -r /app/app/requirements.txt && \
    pip install debugpy

# Aplicar parche a verifier.py dentro del venv
RUN /app/venv/bin/python - <<'PY'
import sysconfig, os, sys
try:
    site = sysconfig.get_paths()['purelib']
except Exception:
    sys.exit(0)

verifier_path = os.path.join(site, 'ask_sdk_webservice_support', 'verifier.py')
if not os.path.exists(verifier_path):
    sys.exit(0)

with open(verifier_path, 'r', encoding='utf-8') as f:
    src = f.read()

needle = (
    '        now = datetime.utcnow()\n'
    '        if not (x509_cert.not_valid_before <= now <=\n'
    '                x509_cert.not_valid_after):\n'
    '            raise VerificationException("Signing Certificate expired")'
)
patch = (
    '        from datetime import timezone\n'
    '        now = datetime.now(timezone.utc)\n'
    "        not_valid_before = getattr(x509_cert, 'not_valid_before_utc', None) or x509_cert.not_valid_before.replace(tzinfo=timezone.utc)\n"
    "        not_valid_after = getattr(x509_cert, 'not_valid_after_utc', None) or x509_cert.not_valid_after.replace(tzinfo=timezone.utc)\n"
    '        if not (not_valid_before <= now <= not_valid_after):\n'
    '            raise VerificationException("Signing Certificate expired")'
)

if needle in src:
    with open(verifier_path, 'w', encoding='utf-8') as f:
        f.write(src.replace(needle, patch))
PY

# Instalar ASK CLI (v2) globalmente
RUN npm install -g ask-cli || true

# Persistir credenciales de ASK CLI en /data/.ask (almacenamiento persistente de Home Assistant)
RUN mkdir -p /data/.ask && ln -s /data/.ask /root/.ask

# Script de arranque que lee /data/options.json de Home Assistant
RUN printf '#!/bin/bash\n\
set -e\n\
if [ -f /data/options.json ]; then\n\
  export SKILL_HOSTNAME=$(jq -r ".SKILL_HOSTNAME // empty" /data/options.json)\n\
  export MA_HOSTNAME=$(jq -r ".MA_HOSTNAME // empty" /data/options.json)\n\
  export APP_USERNAME=$(jq -r ".APP_USERNAME // empty" /data/options.json)\n\
  export APP_PASSWORD=$(jq -r ".APP_PASSWORD // empty" /data/options.json)\n\
  export PORT=$(jq -r ".PORT // 5000" /data/options.json)\n\
  export DEBUG_PORT=$(jq -r ".DEBUG_PORT // 0" /data/options.json)\n\
  export LOCALE=$(jq -r ".LOCALE // \"es-ES\"" /data/options.json)\n\
  export AWS_DEFAULT_REGION=$(jq -r ".AWS_DEFAULT_REGION // \"eu-west-1\"" /data/options.json)\n\
  export TZ=$(jq -r ".TZ // \"Europe/Madrid\"" /data/options.json)\n\
fi\n\
export QUIET_HTTP=1\n\
if [ -n "${DEBUG_PORT}" ] && [ "${DEBUG_PORT}" != "0" ]; then\n\
  exec /app/venv/bin/python -m debugpy --listen 0.0.0.0:${DEBUG_PORT} /app/src/app.py\n\
else\n\
  exec /app/venv/bin/python /app/src/app.py\n\
fi\n' > /app/run.sh && chmod +x /app/run.sh

EXPOSE 5000

CMD ["/app/run.sh"]

