# Instalación y configuración documentadas

Esta guía describe la PoC realizada en Windows con OpenProject Community Edition local. Los comandos y las respuestas del asistente proceden del registro técnico original; las rutas y los identificadores de ejemplo se presentan sin datos privados. No es una receta de despliegue empresarial ni una prueba nueva.

## 1. Preparar OpenProject

El entorno utilizado tenía OpenProject y PostgreSQL en Docker Desktop, con OpenProject accesible en `http://localhost:8080` y la API v3 habilitada. La cuenta de prueba tenía un token personal de API y permisos acordes con cada operación. La documentación original no registra los comandos concretos de creación de los contenedores; por eso no se incluyen aquí.

Comprobaciones de entorno registradas:

```powershell
python --version
git --version
uv --version
py -m pipx --version
docker ps
```

Se comprobó también que OpenProject abría en el navegador. El token se generó en la sección de tokens de la cuenta. Para crear usuarios se requirieron permisos administrativos en OpenProject.

## 2. Instalar el primer servidor MCP

Servidor de terceros: [jtauschl/openproject-ce-mcp](https://github.com/jtauschl/openproject-ce-mcp). Instalación documentada:

```powershell
# Si pipx todavía no está instalado:
py -m pip install --user pipx
py -m pipx ensurepath

# Después de reabrir PowerShell si cambió PATH:
py -m pipx install openproject-ce-mcp
openproject-ce-mcp configure
openproject-ce-mcp --version
```

El registro de instalación menciona la versión `0.4.2` del primer servidor. Una instalación nueva puede resolver otra versión.

## 3. Configurar acceso y permisos

El asistente `configure` se ejecutó dos veces. Los rótulos siguientes son descriptivos y respetan el orden registrado; no son una transcripción literal de la consola.

| Opción | Primera pasada: lectura | Segunda pasada: escritura |
| --- | --- | --- |
| Configuración global | `N` (Enter) | `N` |
| Configuración dentro del proyecto (`project-scoped`) | `Y` | `Y` |
| Cliente Claude | `N` (Enter) | `N` |
| Cliente Codex | `Y` (Enter) | `Y` |
| Cliente VS Code / Copilot | `N` (Enter) | `N` |
| Cliente Cursor | `N` (Enter) | `N` |
| URL de OpenProject | `http://localhost:8080` | `http://localhost:8080` |
| API token | Token personal, omitido aquí | Token personal, omitido aquí |
| Habilitar escritura | `N` (Enter) | `Y` |

En la primera pasada, `OPENPROJECT_READ_PROJECTS = "*"` permitió leer los proyectos visibles para la cuenta; no se habilitaron proyectos ni categorías de escritura. En la segunda, se indicó **un proyecto de prueba** como alcance de escritura. Su ID real se omite deliberadamente; para reproducir la prueba, se usaría el ID de un proyecto propio autorizado.

| Categoría de escritura en la segunda pasada | Respuesta |
| --- | --- |
| Paquetes de trabajo | `Y` |
| Proyectos | `Y` |
| Membresías | `Y` |
| Versiones | `N` |
| Tableros | `Y` |
| Reuniones | `N` |

Un intento inicial de crear otro proyecto encontró una restricción relacionada con el alcance de `WRITE_PROJECTS`. La configuración final conservada en el registro amplió `OPENPROJECT_WRITE_PROJECTS` a `"*"`, con escritura de versiones y reuniones en `"false"`. El comodín fue parte de un **laboratorio local** y no es apropiado como valor predeterminado en producción. Que la categoría «Tableros» estuviera habilitada no demuestra la creación de una vista Kanban visual.

## 4. Conectar el primer MCP con Codex

El asistente registró el servidor con el nombre `openproject`. En la PoC se utilizó `.codex/config.toml` dentro de la carpeta de proyecto confiable abierta en VS Code. La entrada indicó el ejecutable instalado por `pipx`, la URL base y el token. Codex inició el servidor como subproceso local por `stdio`.

El archivo privado de configuración no se publica. El [ejemplo seguro](../examples/config.toml.example) muestra la estructura de ambos servidores con marcadores. El ejecutable exacto de `pipx` depende de la instalación de Windows; debe verificarse localmente antes de completar `command`.

## 5. Instalar el segundo MCP

Servidor de terceros: [gerke-homelab/openproject-mcp-server](https://github.com/gerke-homelab/openproject-mcp-server). Se añadió porque el alta de usuarios no estaba expuesta por el primer servidor en la configuración probada.

```powershell
git clone https://github.com/gerke-homelab/openproject-mcp-server.git C:\OpenProject-MCP-Usuarios
cd C:\OpenProject-MCP-Usuarios
uv sync
Test-Path "C:\OpenProject-MCP-Usuarios\.venv\Scripts\openproject-mcp.exe"
```

`C:\OpenProject-MCP-Usuarios` es una ruta de ejemplo genérica. El registro identifica la versión `4.1.0` en el `pyproject.toml` revisado entonces; `git clone` sin fijar una revisión puede obtener código distinto en el futuro.

## 6. Registrar ambos servidores

El segundo proceso se registró como `openproject_admin`, con `command` apuntando al ejecutable de `.venv` y `cwd` a su carpeta. Usó `OPENPROJECT_URL` y `OPENPROJECT_API_KEY`, a diferencia de `OPENPROJECT_BASE_URL` y `OPENPROJECT_API_TOKEN` del primero. El filtro documentado fue:

```toml
OPENPROJECT_MCP_INCLUDE_TAGS = "create_user,get_user,list_users"
```

Ese filtro exponía las tres herramientas de usuarios necesarias en la PoC. Los permisos reales seguían dependiendo de la cuenta de OpenProject asociada con la clave. El segundo MCP no precisó una consola HTTP abierta: Codex lo ejecutó localmente por `stdio`.

## 7. Verificar las conexiones

Tras editar el TOML, se recargó Codex o la ventana de VS Code y se verificó que aparecieran `openproject` y `openproject_admin`. Las consultas iniciales documentadas fueron identidad (`get_current_user`), proyectos (`list_projects`) y usuarios (`list_users` del segundo MCP). Después se comprobó mediante lecturas cada operación de escritura.

En el relevamiento se observaron aproximadamente 85 herramientas en la configuración inicial de lectura y 127 después de habilitar escritura; esas cifras corresponden al momento de la prueba y pueden variar con las versiones. La verificación se hizo en Codex y OpenProject durante la PoC; esta guía no ejecuta funciones ni realiza nuevas llamadas.

Para un uso empresarial habría que limitar los proyectos autorizados, evitar claves administrativas compartidas, guardar secretos fuera del repositorio, auditar operaciones y revisar las versiones y permisos efectivos de ambos servidores.
