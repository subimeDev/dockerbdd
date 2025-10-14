# dockerbdd

# 🐳 Flask + MySQL con Docker Compose

Proyecto base para levantar una aplicación **Flask** conectada a una base de datos **MySQL** usando **Docker Compose**.  
Funciona perfecto tanto en Linux (Kali, Ubuntu, Debian) como en Windows con WSL o VirtualBox.  

---

## 🚀 Estructura del proyecto
---
lab/
├─ docker-compose.yml
└─ app/
├─ app.py
├─ Dockerfile
└─ requirements.txt
---


---

## ⚙️ Archivos

### 🧩 `docker-compose.yml`
```yaml
version: "3.9"

services:
  mysqldb:
    image: mysql:8.0
    container_name: mysqldb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_DATABASE: "nube"
      MYSQL_USER: "admin"
      MYSQL_PASSWORD: "admin123"
    ports:
      - "3307:3306"
    volumes:
      - db_data:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD-SHELL", "mysql -uroot -p123456 -e 'SELECT 1'"]
      interval: 5s
      timeout: 3s
      retries: 10

  python_app:
    build: ./app
    container_name: python_app
    environment:
      DB_HOST: mysqldb
      DB_USER: admin
      DB_PASS: admin123
      DB_NAME: nube
    depends_on:
      mysqldb:
        condition: service_healthy
    ports:
      - "8000:8000"
    restart: always

volumes:
  db_data:

#app.py

from flask import Flask, jsonify
from sqlalchemy import create_engine, text
import os, time, sys, uuid, datetime

# ===== Config DB =====
DB_HOST = os.getenv("DB_HOST", "mysqldb")
DB_USER = os.getenv("DB_USER", "admin")
DB_PASS = os.getenv("DB_PASS", "admin123")
DB_NAME = os.getenv("DB_NAME", "nube")

CONTAINER_ID = os.popen("hostname").read().strip()
START_TOKEN  = str(uuid.uuid4())
START_TIME   = datetime.datetime.utcnow().isoformat() + "Z"

DB_URL = f"mysql+pymysql://{DB_USER}:{DB_PASS}@{DB_HOST}:3306/{DB_NAME}"

def wait_for_db(max_tries=30, delay=1):
    for i in range(max_tries):
        try:
            engine = create_engine(DB_URL, pool_pre_ping=True)
            with engine.connect() as conn:
                conn.execute(text("SELECT 1"))
            print("[OK] Conectado a la base de datos.")
            return engine
        except Exception as e:
            print(f"[wait_for_db] intento {i+1}/{max_tries} -> BD no lista: {e}")
            time.sleep(delay)
    print("[ERROR] No se pudo conectar a la BD a tiempo.", file=sys.stderr)
    sys.exit(1)

engine = wait_for_db()

# Crear tabla simple de auditoría/clicks
with engine.begin() as conn:
    conn.execute(text("""
        CREATE TABLE IF NOT EXISTS clicks (
            id INT AUTO_INCREMENT PRIMARY KEY,
            timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            container_id VARCHAR(255),
            start_token VARCHAR(255)
        )
    """))

app = Flask(__name__)

@app.get("/")
def index():
    with engine.begin() as conn:
        conn.execute(
            text("INSERT INTO clicks (container_id, start_token) VALUES (:cid, :tok)"),
            {"cid": CONTAINER_ID, "tok": START_TOKEN},
        )
        total = conn.execute(text("SELECT COUNT(*) FROM clicks")).scalar()
    return jsonify({
        "message": "Servidor Flask funcionando dentro de Docker 🎉",
        "total_clicks": total,
        "container_id": CONTAINER_ID,
        "start_token": START_TOKEN,
        "started_at": START_TIME
    })

@app.get("/health")
def health():
    return "OK", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)


requierements.txt

Flask==3.0.0
pymysql==1.1.0
SQLAlchemy==2.0.23



DockerFile

FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]


 contruir y iniciar contenedores
docker compose up -d --build



 bbbb#
