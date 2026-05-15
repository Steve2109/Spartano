1️⃣ Preparación del sistema operativo
# 1. Actualizar el sistema
sudo apt update && sudo apt upgrade -y

# 2. Instalar dependencias básicas
sudo apt install -y \
    curl wget gnupg2 lsb-release \
    software-properties-common \
    build-essential \
    python3-pip python3-venv python3-dev \
    libpq-dev \
    nginx \
    fail2ban \
    ufw

# 3. Crear un usuario dedicado para la aplicación
sudo adduser --system --group --home /opt/contaec contaec
sudo usermod -aG sudo contaec   # solo si necesitas sudo dentro del LXC para tareas de admin

# 4. Cambia a ese usuario para el resto de la instalación:
sudo -i -u contaec
cd /opt/contaec

2️⃣ Instalación de PostgreSQL
# 1. Salir del usuario contaec si estás dentro y volver como root para instalar PG
exit
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql

# 2. Crear base de datos y usuario de la aplicación
sudo -u postgres psql <<EOF
-- Crear rol de aplicación
CREATE ROLE contaec_user WITH
    LOGIN
    PASSWORD 'StrongDbPassword!2025'
    NOSUPERUSER
    INHERIT
    NOCREATEDB
    NOCREATEROLE
    NOREPLICATION;

-- Crear base de datos
CREATE DATABASE contaec_db
    WITH OWNER = contaec_user
         ENCODING = 'UTF8'
         LC_COLLATE = 'en_US.utf8'
         LC_CTYPE = 'en_US.utf8'
         TEMPLATE = template0;

-- Conceder privilegios
GRANT ALL PRIVILEGES ON DATABASE contaec_db TO contaec_user;
\q
EOF

## Nota de seguridad – Cambia StrongDbPassword!2025 por una contraseña larga y aleatoria. Nunca la dejes en el código; irá al .env (ver paso 4). ##

# 3. Endurecer PostgreSQL
##Edita /etc/postgresql/15/main/pg_hba.conf y asegúrate de que las conexiones locales usen md5 (o mejor scram-sha-256 si tu PG lo soporta) y que no haya líneas que permitan acceso desde cualquier IP sin contraseña.##
##Luego recarga:##
sudo systemctl reload postgresql

3️⃣ Creación del entorno virtual y dependencias de FastAPI
# Volver como usuario contaec
sudo -i -u contaec
cd /opt/contaec

# Crear venv
python3 -m venv venv
source venv/bin/activate

# Actualizar pip
pip install --upgrade pip

# Instalar FastAPI, Uvicorn y dependencias de base
pip install \
    fastapi[all] \
    uvicorn[standard] \
    sqlalchemy \
    psycopg2-binary \
    alembic \
    python-dotenv \
    passlib[bcrypt] \
    python-jose[cryptography] \
    python-multipart \
    email-validator \
    pydantic[email] \
    python-magic \
    # Para limitación de tasa (usaremos slowapi)
    slowapi \
    # Para validación y sanitizado de entradas (pydantic ya lo hace, pero añadimos bleach para HTML)
    bleach \
    # Para manejo de archivos ZIP/CSV/Excel (pandas opcional, pero liviano)
    pandas openpyxl xlrd \
    # Para generación de PDF (reportlab) – se usará más adelante
    reportlab
#Tip de ligereza – Si en algún momento notas que el consumo de RAM es alto, puedes reemplazar pandas por openpyxl + csv puro para la fase de import/export. Por ahora lo dejamos por facilidad de desarrollo.

# 4️⃣ Estructura del proyecto (esqueleto)
/opt/contaec
│
├─ app/
│   ├─ __init__.py
│   ├─ main.py               # punto de entrada de FastAPI
│   ├─ core/
│   │   ├─ config.py         # carga de .env y settings
│   │   ├─ security.py       # hash de pwd, JWT, etc.
│   │   └─ limiter.py        # configuración de slowapi
│   │
│   ├─ db/
│   │   ├─ base.py           # Base declarativa de SQLAlchemy
│   │   └─ session.py        # engine y SessionLocal
│   │
│   ├─ api/
│   │   ├─ deps.py           # dependencias (DB, user, etc.)
│   │   ├─ v1/
│   │   │   ├─ __init__.py
│   │   │   ├─ auth.py       # login, registro, refresh
│   │   │   ├─ users.py      # CRUD de usuarios (multiempresa)
│   │   │   ├─ companies.py  # gestión de empresas
│   │   │   └─ health.py     # endpoint /health
│   │   └─ __init__.py
│   │
│   ├─ models/
│   │   ├─ user.py
│   │   ├─ company.py
│   │   └─ ... (se irán añadiendo en fases posteriores)
│   │
│   ├─ schemas/
│   │   ├─ user.py
│   │   ├─ company.py
│   │   └─ ...
│   │
│   └─ utils/
│       ├─ sanitize.py       # funciones de bleach + validación de tipos
│       └─ clamav.py         # wrapper para clamdscan (se usará en fase 2)
│
├─ migrations/               # Alembic (se crea después)
├─ .env                      # variables de entorno (ver paso 5)
├─ requirements.txt          # freeze de dependencias (opcional)
└─ run.sh                    # script de arranque (para systemd)

# Crear los directorios y archivos básicos
mkdir -p app/{core,db,api/v1,models,schemas,utils}
touch app/__init__.py
touch app/main.py
touch app/core/{config.py,security.py,limiter.py}
touch app/db/{base.py,session.py}
touch app/api/{deps.py,__init__.py}
touch app/api/v1/{__init__.py,auth.py,users.py,companies.py,health.py}
touch app/models/{user.py,company.py}
touch app/schemas/{user.py,company.py}
touch app/utils/{sanitize.py,clamav.py}
touch .env
touch run.sh

# 5️⃣ Archivo .env (variables de entorno maestro)
#Protección del .env
chmod 600 .env          # solo el usuario contaec puede leer
chown contaec:contaec .env

# 6️⃣ Configuración de la aplicación (core/config.py)
# app/core/config.py
from pydantic import BaseSettings, PostgresDsn, Field
from typing import Optional, List
import secrets

class Settings(BaseSettings):
    # ----- DB -----
    POSTGRES_SERVER: str = Field(...)
    POSTGRES_PORT: int = Field(5432)
    POSTGRES_DB: str = Field(...)
    POSTGRES_USER: str = Field(...)
    POSTGRES_PASSWORD: str = Field(...)

    @property
    def SQLALCHEMY_DATABASE_URI(self) -> PostgresDsn:
        return PostgresDsn.build(
            scheme="postgresql+psycopg2",
            user=self.POSTGRES_USER,
            password=self.POSTGRES_PASSWORD,
            host=self.POSTGRES_SERVER,
            port=str(self.POSTGRES_PORT),
            path=self.POSTGRES_DB,
        )

    # ----- JWT -----
    SECRET_KEY: str = Field(default_factory=lambda: secrets.token_urlsafe(64))
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    # ----- Seguridad de configuración por usuario -----
    USER_CONFIG_ENC_KEY: str = Field(...)   # 32‑bytes base64 o raw

    # ----- Rate limiting -----
    RATE_LIMIT_PER_MINUTE: int = 120

    # ----- ClamAV -----
    CLAMD_HOST: str = "127.0.0.1"
    CLAMD_PORT: int = 3310

    # ----- VirusTotal (opcional) -----
    VT_API_KEY: Optional[str] = None

    # ----- General -----
    ENVIRONMENT: str = "production"
    LOG_LEVEL: str = "info"

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
