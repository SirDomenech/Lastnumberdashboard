# Índice de impactos ACB 26/27

Dashboard de estimación de impacto de jugadores y equipos de la Liga Endesa
26/27 — Last Number Data / N3XT Sports. Combina impacto individual e impacto
colectivo en un rating 0-100 por jugador y por equipo, con ranking de
jugadores, ranking de equipos y matriz de arquetipos.

- **Ver el dashboard**: abre `frontend/dashboard/dist/dashboard_roster_final.html`
  en cualquier navegador (fichero único, sin dependencias externas salvo
  Google Fonts) — es el mismo fichero que se publica como Artifact.
- **Cómo funciona el modelo**: ver [`MODELO.md`](MODELO.md).
- **Datos**: `backend/data/` — CSV/JSON/XLSX de entrada y salida del pipeline
  (ver detalle más abajo).

Repo separado en `backend/` (pipeline de datos) y `frontend/` (dashboard),
para que se puedan tocar por separado.

```
backend/
  pipeline/   34 scripts .py, en orden de ejecucion (01_... -> 29_...) mas
              varios scripts de soporte (build_*.py, append_new_31.py).
              29_valencia_dashboard_data.py es el script de produccion
              vigente -- genera backend/data/roster_dashboard_data.json,
              la fuente de datos del dashboard. Los scripts 26/27/28 son
              versiones anteriores del modelo de rating, ya superadas.
  data/       todos los CSV/JSON/XLSX que leen y escriben los scripts
              (inputs y outputs intermedios), mas:
                logos_uploads/  imagenes de referencia subidas en su
                                 momento, sin uso en el pipeline.
                raw_uploads/     los 4 ficheros "en bruto" (subidos por
                                 Miguel durante el trabajo con Claude) que
                                 lee directamente 29_valencia_dashboard_data.py:
                                 Arquetipos_26-27_revision_Miguel.csv,
                                 Jugadores_dashboard_26-27_auditoria.csv,
                                 Box_Scores_ACB.csv y Fichajes_ACB_Players.csv.
                                 Se copiaron aqui (28/09/2026, paquete para
                                 GitHub) y el script se actualizo para leerlos
                                 por ruta relativa, en vez de la ruta absoluta
                                 del contenedor donde se subieron -- asi el
                                 pipeline se puede clonar y ejecutar en
                                 cualquier maquina sin tocar nada.

frontend/
  dashboard/  el dashboard partido en ficheros normales:
                index.html   estructura (abre directo en el navegador)
                styles.css   todo el CSS
                app.js       toda la logica / interactividad
                assets/      logo_acb.png, logo_lnd.png
                build.py     compila lo anterior + backend/data/roster_dashboard_data.json
                             en un unico HTML autocontenido:
                dist/dashboard_roster_final.html
                             <- este es el fichero que se publica como
                             Artifact en claude.ai (tiene que ser un solo
                             HTML, sin dependencias externas salvo Google
                             Fonts).
  archive/    dashboards y artefactos de versiones anteriores del proyecto
              (dashboard.html, dashboard_final.html, dashboard_v2_work.html,
              capturas de pantalla), sin uso ya.
```

## Cómo se actualiza el dashboard

1. Correr el pipeline de backend (o el/los scripts que correspondan según
   qué cambió) desde `backend/pipeline/` -- cada script ya resuelve sus
   rutas relativas contra `backend/data/` automáticamente. El script clave
   es `29_valencia_dashboard_data.py`, que deja el resultado en
   `backend/data/roster_dashboard_data.json`.
2. Editar `frontend/dashboard/index.html` / `styles.css` / `app.js` si hay
   cambios de maqueta, estilo o interactividad.
3. Desde `frontend/dashboard/`, correr `python3 build.py` -- regenera
   `dist/dashboard_roster_final.html`.
4. Publicar `dist/dashboard_roster_final.html` como Artifact.

Nota: `index.html` se puede abrir directo en el navegador para revisar
maqueta/estilos, pero sin pasar por `build.py` no tendrá datos reales (la
variable `DATA` queda sin definir) ni los logos incrustados.

## Requisitos para correr el pipeline

Python 3 + `pandas`, `numpy`, `scipy`. Sin entorno virtual predefinido en el
repo -- `pip install pandas numpy scipy` es suficiente. `29_valencia_dashboard_data.py`
se probó de punta a punta con las rutas relativas de este repo y reproduce
`roster_dashboard_data.json` sin tocar nada más.

Los scripts `01_...` a `28_...` (iteraciones anteriores del modelo, ya
superadas por `29_...`) no se tocaron en esta reorganización: varios todavía
leen ficheros por la ruta absoluta de subida original de la conversación con
Claude y no están pensados para volver a ejecutarse -- se conservan solo como
historial de cómo se llegó al modelo actual. Si algún día hace falta revivir
alguno, hay que localizar el CSV/XLSX que pide y apuntarlo a una ruta local.

## Datos y confidencialidad

`backend/data/` incluye CSV/JSON/XLSX con estadísticas de jugadores y
fichajes recopiladas durante el proyecto (boxscores ACB 25/26, fichero de
fichajes detallado, arquetipos revisados a mano por Miguel, etc.). Antes de
hacer público el repo -- aunque sea solo para el equipo -- conviene decidir
si algún fichero de `backend/data/` debe quedar fuera (por ejemplo si
`Fichajes_ACB_Players.csv` o el boxscore tienen alguna restricción de uso del
proveedor de datos). Ninguno de estos ficheros se ha filtrado ni verificado
para republicación externa.

## Licencia / crédito

Modelo y dashboard: Last Number Data, para N3XT Sports. Sin licencia abierta
definida todavía -- uso interno del equipo hasta que se decida lo contrario.
