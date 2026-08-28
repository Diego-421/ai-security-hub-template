# Guía Técnica — AI Security Hub

Guía de referencia con el setup del entorno y las 7 fases de construcción del proyecto. Cada fase incluye qué se instala/configura, comandos clave y checklist de salida antes de pasar a la siguiente.

---

## Fase 0 — Setup del entorno

### 0.1 Editor e IA local

```bash
# VS Code
https://code.visualstudio.com/download


```

**Extensiones recomendadas en VS Code:** Python, Docker, GitLens, Thunder Client (probar tu API), ESLint, Prettier.

```bash
# Ollama (IA local)
curl -fsSL https://ollama.com/install.sh | sh   # Linux/Mac
# Windows: https://ollama.com/download

ollama pull llama3
ollama pull deepseek-coder   # bueno para análisis de código
ollama run llama3            # probar
```

### 0.2 Git, GitHub CLI y Docker

```bash
git --version
sudo apt install git -y      # Linux
brew install git             # Mac

brew install gh              # Mac
sudo apt install gh -y       # Linux
gh auth login
```

```bash
# Docker Desktop
https://www.docker.com/products/docker-desktop/

docker --version
docker compose version
```

### 0.3 Repo en GitHub — estructura monorepo

```bash
gh repo create ai-security-hub --public --clone
cd ai-security-hub
```

```
ai-security-hub/
├── frontend/
├── backend/
├── infra/
│   └── docker-compose.yml
└── README.md
```

```bash
git checkout -b develop
git push -u origin develop
```

### 0.4 docker-compose.yml — servicios base

Servicios a levantar: **Postgres+pgvector**, **Redis**, **n8n**.

| Servicio | Imagen | Puerto | Notas |
|---|---|---|---|
| postgres | `pgvector/pgvector:pg16` | `5432:5432` | vars `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`; volumen `pgdata:/var/lib/postgresql/data` |
| redis | `redis:7-alpine` | `6379:6379` | — |
| n8n | `n8nio/n8n` | `5678:5678` | `N8N_BASIC_AUTH_ACTIVE=true` |

```bash
mkdir -p infra && cd infra
touch docker-compose.yml   # arma el YAML con la tabla de arriba

docker compose up -d
docker compose ps      # verificar que estén "healthy"
```

Verificación:
- n8n → http://localhost:5678
- Postgres → `psql` o DBeaver a `localhost:5432`

**✅ Checklist de salida:** repo creado con rama `develop`, `docker compose ps` muestra los 3 servicios healthy, Ollama responde a `ollama run llama3`.

---

## Fase 1 — Backend core (FastAPI)

```bash
mkdir backend && cd backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install fastapi uvicorn sqlalchemy alembic psycopg2-binary python-jose passlib pytest httpx
pip freeze > requirements.txt
```

```
backend/
├── app/
│   ├── main.py
│   ├── models/
│   ├── routers/
│   ├── core/        # config, security
│   └── db/
├── alembic/
├── tests/
└── requirements.txt
```

**Qué construir:**
- Autenticación JWT + RBAC (`python-jose`, `passlib`)
- Endpoints para recibir logs/código
- Conexión a Postgres vía SQLAlchemy, migraciones con Alembic
- Tests con Pytest desde el primer endpoint, no al final

```bash
uvicorn app.main:app --reload   # localhost:8000/docs
```

**✅ Checklist de salida:** login con JWT funcionando, al menos 1 migración de Alembic aplicada, tests de auth pasando en CI local.

---

## Fase 2 — Base de datos y embeddings

**Esquema a diseñar:** `usuarios`, `auditorías`, `alertas`, `embeddings` (columna `vector` con pgvector).

**Flujo a implementar:**
1. Endpoint recibe código/logs.
2. Genera embeddings (vía Ollama o `sentence-transformers`).
3. Guarda el vector en Postgres/pgvector.
4. Endpoint de búsqueda semántica (similaridad de coseno) para encontrar incidentes similares.

```bash
pip install sentence-transformers   # alternativa local a Ollama para embeddings
```

**✅ Checklist de salida:** puedes insertar un log, se genera su embedding, y una búsqueda por similitud devuelve resultados coherentes.

---

## Fase 3 — Automatización con n8n

n8n ya corre en Docker desde la Fase 0. Aquí se conecta con el backend y con Ollama.

**Workflow a crear:**
1. **Webhook trigger** — escucha eventos del backend (ej. nueva alerta).
2. **Filtro** — por severidad (ej. solo `high`/`critical` siguen el flujo).
3. **Nodo HTTP → Ollama** — pide una recomendación en lenguaje natural sobre el incidente.
4. **Notificación** — nodo de Telegram o Discord con el resumen + recomendación.

Esto es lo que diferencia el proyecto de un CRUD normal: la orquestación IA + notificación automática.

**✅ Checklist de salida:** un evento de prueba enviado por webhook dispara el filtro, llega a Ollama y termina como mensaje en Telegram/Discord.

---

## Fase 4 — Frontend (React + Vite + Tailwind)

```bash
npm create vite@latest frontend -- --template react
cd frontend
npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
npm install axios react-query   # o @tanstack/react-query
npm run dev
```

**Pantallas a construir:**
- Login
- Vista de alertas en tiempo real (WebSockets o polling)
- Formulario para subir código/logs
- Panel de métricas (Recharts)

**Estado async:** consumir la API con React Query (`@tanstack/react-query`), no `useEffect` manual.

**✅ Checklist de salida:** login conectado al backend real, alertas se actualizan sin recargar la página, gráfico de métricas renderizando datos reales.

---

## Fase 5 — Seguridad DevSecOps

```bash
# Trivy
brew install trivy              # Mac
sudo apt install trivy -y       # Linux (puede requerir repo adicional)

trivy image nombre_de_tu_imagen   # escanear imagen Docker
trivy fs .                        # escanear dependencias del proyecto
```

**Pipeline de GitHub Actions:**

```
lint (ruff / eslint) → tests (pytest / vitest) → Trivy scan de imágenes Docker → SonarQube (opcional, self-hosted)
```

Documenta el pipeline en el `README.md` — es lo primero que revisan reclutadores técnicos.

**✅ Checklist de salida:** un push a `develop` dispara el pipeline completo y falla correctamente si algo no pasa lint/tests/scan.

---

## Fase 6 — Observabilidad y pulido final

- **Prometheus + Grafana** para métricas del sistema (contenedores en `infra/docker-compose.yml`).
- **Documentación técnica**: arquitectura, decisiones de diseño, diagramas con Mermaid.
- **Video demo corto** para LinkedIn — vale más que 1000 líneas de README.

**✅ Checklist de salida:** dashboard de Grafana mostrando métricas reales, README con diagrama de arquitectura, video demo grabado y publicado.

---

## Flujo de trabajo en Git (transversal a todas las fases)

```bash
# Por cada feature
git checkout -b feature/nombre-feature
# ... commits ...
git push -u origin feature/nombre-feature
gh pr create --base develop --head feature/nombre-feature
```

**Conventional Commits:**

```bash
git commit -m "feat: agregar docker-compose con postgres, redis y n8n"
git commit -m "chore: configurar .gitignore para python y node"
git commit -m "docs: agregar guía técnica de instalación"
```

**Al cerrar una fase:**

```bash
git checkout main
git merge develop
git tag -a v0.1.0 -m "Fase 0: entorno y docker-compose funcional"
git push origin main --tags

gh release create v0.1.0 --notes "Setup inicial: Docker, Postgres+pgvector, Redis, n8n levantados y funcionando."
```

**Tablero de progreso:**

```bash
gh project create --owner @me --title "AI Security Hub"
# o desde GitHub web: Repo → Projects → New Project → plantilla "Board"
```
