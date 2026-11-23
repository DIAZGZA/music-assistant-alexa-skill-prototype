FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Instalar Python 3.12, dependencias del sistema y Node.js 20 LTS
RUN apt-get update && \
    apt-get install -y python3 python3-venv python3-pip libssl-dev curl gnupg ca-certificates && \
    curl -fsSL https://nodesource.com | bash - && \
    apt-get install -y nodejs && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copiar requirements e instalar dependencias en entorno virtual
COPY app/requirements.txt /app/requirements.txt
RUN python3 -m venv venv && \
    . venv/bin/activate && \
    pip install --upgrade pip && \
    pip install -r requirements.txt && \
    pip install debugpy

# Aplicar parche a verifier.py dentro del venv
RUN /app/venv/bin/python - <<'PY'
import sysconfig, os, sys
try:
        site = sysconfig.get_paths()['purelib']
except Exception:
        print('Could not determine site-packages path; skipping verifier patch')
        sys.exit(0)

verifier_path = os.path.join(site, 'ask_sdk_webservice_support', 'verifier.py')
if not os.path.exists(verifier_path):
        print('verifier.py not found at', verifier_path, '; skipping patch')
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
    '        # Use timezone-aware UTC datetimes and updated cryptography properties\n'
    "        not_valid_before = getattr(x509_cert, 'not_valid_before_utc', None) or x509_cert.not_valid_before.replace(tzinfo=timezone.utc)\n"
    "        not_valid_after = getattr(x509_cert, 'not_valid_after_utc', None) or x509_cert.not_valid_after.replace(tzinfo=timezone.utc)\n"
    '        if not (not_valid_before <= now <= not_valid_after):\n'
    '            raise VerificationException("Signing Certificate expired")'
)

if needle in src:
    new_src = src.replace(needle, patch)
    backup = verifier_path + '.orig'
    try:
        if not os.path.exists(backup):
            with open(backup, 'w', encoding='utf-8') as b:
                b.write(src)
        with open(verifier_path, 'w', encoding='utf-8') as f:
            f.write(new_src)
        print('Patched', verifier_path, '(backup at', backup + ')')
    except Exception as e:
        print('Failed to write patch:', e)
else:
    print('No patch needed for verifier.py')
PY

# Instalar ASK CLI (v2) globalmente
RUN npm install -g ask-cli || true

# Copiar código fuente, manifiestos, assets y scripts
COPY app /app/src
COPY app /app/app
COPY assets /app/assets
COPY scripts /app/scripts
RUN chmod +x /app/scripts/ask_create_skill.sh || true

# Configuración del entorno
ENV AWS_DEFAULT_REGION=us-east-1
ENV TZ=UTC
ENV MA_HOSTNAME=""
ENV SKILL_HOSTNAME=""
ENV PORT=5000
ENV LOCALE=en-US
ENV QUIET_HTTP=1

ARG DEBUG_PORT=0
ENV DEBUG_PORT=${DEBUG_PORT}

EXPOSE ${PORT}

CMD ["/bin/sh", "-lc", "if [ -n \"${DEBUG_PORT}\" ] && [ \"${DEBUG_PORT}\" != \"0\" ]; then exec /app/venv/bin/python -m debugpy --listen 0.0.0.0:${DEBUG_PORT} src/app.py; else exec /app/venv/bin/python src/app.py; fi"]
