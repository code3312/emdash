# Despliegue de EmDash en Cloudflare

Este documento detalla el proceso para levantar una nueva instancia de EmDash en la plataforma de Cloudflare y explica por qué los despliegues estándar de Pages no son viables.

> [!WARNING]  
> **Importante:** El proyecto `demos/cloudflare` de EmDash está diseñado para ejecutarse exclusivamente como un **Cloudflare Worker**. No debes usar **Cloudflare Pages** para alojar este tipo de instancia.

## 1. El Problema con Cloudflare Pages (Error 404)

Si intentas desplegar EmDash vinculando el repositorio automatizado a Cloudflare Pages (vía integraciones de GitHub o GitLab), la compilación informará que se procesaron los archivos exitosamente, pero al navegar a tu URL de producción el resultado será un error 404 con una pantalla propia de Cloudflare.

**¿Por qué sucede esto?**
- Pages de Cloudflare simplemente toma los activos compilados (HTML estático, CSS, JavaScript) en la carpeta de salida (por lo general alrededor de 250+ archivos) y los sirve.
- Al no generar Astro un `index.html` de base, y debido a que el entorno subyacente de Pages ignora el _entrypoint_ SSR que tiene el proyecto en su backend (`src/worker.ts`), el despliegue termina sin un gestor de rutas válido.
- EmDash requiere funcionalidades técnicas avanzadas que residen un nivel más profundo. Específicamente, exporta la clase `PluginBridge` a través de **Worker Loaders** (RPC) para inicializar plugins aislados bajo un ecosistema "sandbox". Cloudflare Pages tiene ciertas limitaciones manejando estos entrypoints propios, mientras que Cloudflare Workers está especialmente habilitado para manejarlos.

---

## 2. Proceso de Despliegue Correcto (Cloudflare Workers)

Para alojar EmDash, el CLI (Terminal) será tu principal motor para aprovisionar y publicar tu arquitectura completa vía Cloudflare Workers. 

> [!NOTE]  
> Debes tener autorización previa de Cloudflare en tu máquina. Si no la tienes, ejecuta `npx wrangler login` primero.

### Paso 1: Configuración Inicial de la Base de Datos D1

EmDash demanda una base de datos pre-configurada (usa SQL vía `Kysely` conectada contra una instancia de Cloudflare D1). Para crearla, sitúate en la raíz del proyecto y corre el siguiente script:

```bash
pnpm --filter @emdash-cms/demo-cloudflare db:create
```

La consola te imprimirá un mensaje indicando que se ha creado la base de datos `emdash-demo`, acompañado de un bloque de texto que incluye un **ID (database_id)**. 

### Paso 2: Vincular la Base de Datos

Copia el ID generado en el paso anterior. Accede al código fuente del demo en `demos/cloudflare/wrangler.jsonc` y localiza el arreglo de `d1_databases`. Reemplaza o adiciona el `database_id`:

```jsonc
"d1_databases": [
  {
    "binding": "DB",
    "database_name": "emdash-demo",
    "database_id": "INGRESA_TU_DATABASE_ID_AQUI"
  }
]
```

### Paso 3: Lanzar el Despliegue (Build & Deploy)

Al emplear la arquitectura completa, delegamos en Wrangler la responsabilidad de inyectar el Entrypoint, interpretar los Worker Loaders y subir tanto estáticos de front como dependencias del worker usando un único comando:

```bash
pnpm --filter @emdash-cms/demo-cloudflare run deploy
```

> **Nota Interna:** Este comando empaqueta y verifica los tipos en tu monorepo ejecutando el script nativo  (`pnpm build:all && wrangler deploy`).

### Paso 4: Finalización (Migraciones al Vuelo)

Wrangler indicará los URL donde se ha publicado el servicio (`<proyecto>.<tu_usuario>.workers.dev`). 

No es necesario generar o correr comandos de migración. EmDash detectará que su base de datos `emdash-demo` no tiene tablas (huérfana) al recibir la primera petición externa, y ejecutará las rutinas de SQLite de forma autónoma.
