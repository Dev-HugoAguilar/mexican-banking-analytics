# Banca múltiple mexicana · Comparativo financiero

Explora la participación, el crecimiento y la rentabilidad de los bancos en
México con cifras publicadas por la **CNBV**: ranking por institución, evolución
temporal, dispersión ROA–ROE y tabla de detalle, todo con un mismo estado de
filtros.

[![Build](https://github.com/Dev-HugoAguilar/mexican-banking-analytics/actions/workflows/tests.yml/badge.svg)](https://github.com/Dev-HugoAguilar/mexican-banking-analytics/actions/workflows/tests.yml)
[![React](https://img.shields.io/badge/React-TypeScript-3178C6?logo=react&logoColor=white)](app)
[![Datos](https://img.shields.io/badge/datos-CNBV-b7791f)](#datos-y-atribución)

> **Alcance.** Aplicación de lectura del boletín estadístico. No es un sistema de
> riesgo ni un modelo de solvencia: compara cifras publicadas y declara sus
> unidades.

---

## Ejecutar

Aplicación React + Vite sin backend propio: consume la API compartida
[`portfolio-api`](https://github.com/Dev-HugoAguilar/portfolio-api) (FastAPI + MongoDB)
en el puerto **8010**. La API no se copia en este repositorio — se clona aparte —
para no duplicar código ni datos.

### 1. Requisitos

- **Python 3.12** para la API.
- **Node.js 20 o superior** con `npm` (verificado con Node 24.11.0 y npm 11.6.1).
- **MongoDB**, Atlas o local: solo hace falta la URI de conexión.

### 2. API en el puerto 8010

```bash
git clone https://github.com/Dev-HugoAguilar/portfolio-api.git
cd portfolio-api
python -m venv .venv
```

Activa el entorno virtual:

```bash
# Windows (PowerShell o CMD)
.venv\Scripts\activate
# Linux / macOS
source .venv/bin/activate
```

Instala y arranca:

```bash
pip install -r requirements-dev.txt
cp .env.example .env        # Windows PowerShell: Copy-Item .env.example .env
python -m uvicorn app.main:app --port 8010
```

Variables que hay que rellenar en `.env` (no se versiona):

| Variable | Qué poner |
|---|---|
| `MONGODB_URI` | **Obligatoria.** URI de conexión de MongoDB: cadena de Atlas (`mongodb+srv://…`, con el usuario y la clave del clúster) o local (`mongodb://127.0.0.1:27017/hugo_portfolio`). |
| `MONGODB_DB` | Base donde están las colecciones publicadas (`hugo_portfolio`). |
| `ALLOWED_ORIGINS` | Orígenes permitidos por CORS; ver el paso 3. |
| `MONGO_ROOT_*`, `APP_DB_*`, `BACKEND_HOST_PORT` | Solo si levantas MongoDB/API con Docker Compose. |
| `MARKETING_GCP_PROJECT` | Solo para volver a extraer datos de GA4 desde BigQuery; la API y el tablero leen de MongoDB. |

`/health/ready` comprueba proceso **y** MongoDB:

```bash
curl http://127.0.0.1:8010/api/v1/health/ready
# {"status":"ready","mongo":"ok","version":"1.0.0"}
```

### 3. Aplicación en el puerto 5173

```bash
cd app
npm ci
npm run dev
```

Vite sirve en <http://127.0.0.1:5173>. El cliente consulta la API en
`http://127.0.0.1:8010/api/v1/cnbv/banca-multiple` (ver `app/src/api/client.ts`),
así que la petición es **de otro origen** (5173 → 8010).

**CORS.** Si la API no autoriza el origen del dev server, el navegador bloquea la
respuesta: error de CORS en la consola y el tablero se queda sin datos. Añade los
dos orígenes de desarrollo a `ALLOWED_ORIGINS` en el `.env` de la API y reinicia
uvicorn:

```env
ALLOWED_ORIGINS=http://localhost:8000,http://127.0.0.1:8000,http://127.0.0.1:5173,http://localhost:5173
```

La lista va separada por comas, sin espacios.

### 4. Compilación de producción

```bash
npm run build
```

Ejecuta `tsc --noEmit`, `vite build` y el minificador de `app/tools/`. La salida
queda en **`app/dist/`** (`app.js` y `app.css`), el módulo que el portafolio carga
con `import()`.

### 5. Comprobación funcional

Endpoint real de este tablero (definido en `app/api/v1/routes/cnbv.py` de la API):

```bash
curl http://127.0.0.1:8010/api/v1/cnbv/banca-multiple/catalogo
```

Debe responder `HTTP 200` con `fuente`, `boletines`, `periodos`, `conceptos`,
`instituciones`, `indicadores` y `cobertura`. Obtenido al verificar este
repositorio: `cobertura.periodos_importados = 7`, `periodo_min = "2023-10"`,
`periodo_max = "2024-10"` y `hechos = 56248`. Si la base no tiene boletines, el
endpoint responde `404` con el mensaje que indica el comando exacto para
cargarlos: `python -m scripts.importar_cnbv --descargar`.

Con la API en marcha, abre <http://127.0.0.1:5173>: ranking, serie, dispersión y
tabla deben responder a la misma selección.

> **Versión compatible:** probado con `portfolio-api` en el commit `8a27227`.

### 6. Pruebas de navegador (manuales, fuera de CI)

`app/tests/e2e/` usa Playwright con el Chrome instalado en el equipo
(`channel: 'chrome'`, sin descargar navegadores). **No se ejecuta en CI**: necesita
la API real en 8010, el tablero servido por HTTP y Chrome instalado, y
`portafolio.spec.ts` recorre además el sitio del portafolio.

```bash
npm run test:e2e
```

![Portada del comparativo bancario](docs/cover.jpg)

## Qué demuestra

- **Un sector comparable sin trampas**: la columna «Sistema» es el agregado
  publicado y **nunca** se suma con las instituciones.
- **Flujos contra su propia base**: los importes acumulados solo se comparan con
  el mismo corte del año anterior.
- **Huecos, no interpolaciones**: los meses sin boletín se marcan como faltantes.
- **Consistencia de estado**: 66 pruebas de navegador verifican que ranking,
  serie, dispersión y tabla responden a la misma selección, en escritorio y móvil.

## Datos y atribución

**CNBV**, Boletín Estadístico de Banca Múltiple (sector 40 del Portafolio de
Información): 7 boletines importados, **56 248 hechos**, **158 conceptos**,
**50 instituciones** y 7 periodos disponibles (oct 2023 – oct 2024), conciliados
celda por celda contra los archivos originales sin discrepancias.

## Estructura

```text
app/                 Aplicación React + TypeScript (Vite)
  src/               Estado, consultas, tablas y visualizaciones D3
  tests/             Pruebas de navegador con Playwright
  tools/             Minificador y utilidades de build
docs/LEEME.md        Documentación original del proyecto
docs/cover.jpg       Captura real de la aplicación
```

## Límites

- Solo banca múltiple: no incluye banca de desarrollo, sofomes ni otras entidades.
- Los conceptos acumulados no son comparables con saldos puntuales.
- La cobertura temporal depende de los boletines publicados (7 periodos).
- Las cifras son nominales y no están ajustadas por inflación.

## Seguridad y contribución

Sin credenciales ni datos personales en el repositorio. Ver [`SECURITY.md`](SECURITY.md)
y [`CONTRIBUTING.md`](CONTRIBUTING.md).
