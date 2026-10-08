# Registro de decisiones — Proyecto Steam Data Pipeline

Este documento registra las decisiones técnicas y de arquitectura tomadas a lo largo del proyecto, con el razonamiento detrás de cada una. El objetivo es doble: (1) que este proceso de pensamiento quede documentado para futuras referencias propias, y (2) reducir la dependencia de asistencia externa en el próximo proyecto, teniendo un registro de *por qué* se tomó cada camino, no solo *qué* se hizo.

Cada entrada sigue el formato: **Qué se decidió** / **Por qué** / **Alternativas descartadas**.

---

## 2026-09-16 — Alcance y gancho del proyecto

**Qué se decidió:** Proyecto individual de portafolio sobre datos de Steam/videojuegos, con el gancho: "¿qué predice si un juego indie sobrevive su primer año?"

**Por qué:** El dominio de videojuegos combina interés personal genuino con un ángulo de inteligencia de mercado real (la industria mueve más dinero que cine y música juntos), lo que le da al insight potencial impacto profesional, no solo curiosidad de fan. Se buscó un gancho con pregunta clara y sorpresa potencial, siguiendo el criterio de que el proyecto debe resolver una pregunta real, no solo explorar datos.

**Alternativas descartadas:** Otras ideas de dominio evaluadas en el proyecto de equipo (SICOP, compras públicas de Costa Rica) se mantienen separadas — este es un proyecto individual, sin mezclar contexto.

---

## 2026-09-16 — Fuentes de datos: Steam Store API + SteamSpy

**Qué se decidió:** Usar la Steam Store API (`store.steampowered.com/api/appdetails`) para metadata oficial (género, fecha de lanzamiento, precio) y SteamSpy (`steamspy.com/api.php`) para estimados de dueños y jugadores concurrentes.

**Por qué:** Ambas se probaron en vivo (con Counter-Strike 2, appid 730) antes de comprometerse a usarlas — no solo se confió en documentación de terceros. Ninguna requiere autenticación para consultas básicas.

**Limitación documentada:** SteamSpy es mantenida por un tercero, no por Valve, y se actualiza una vez al día — no es un dato "oficial" y su estabilidad no está garantizada (ver entrada del 2026-09-23 sobre el endpoint de género).

---

## 2026-09-16 — Arquitectura: medallion + Delta Lake en Databricks

**Qué se decidió:** Usar arquitectura medallion (bronze/silver/gold) sobre Delta Lake en Databricks, con GitHub como fuente de verdad del código conectado vía Repos.

**Por qué:** Es el mismo patrón usado en el proyecto de referencia que inspiró este trabajo (pipeline de vuelos de EE.UU. de Iván Lauer), y es el estándar de la industria para pipelines de datos reproducibles: bronze guarda datos crudos sin transformar, silver limpia y normaliza, gold modela para consumo (ej. star schema para análisis).

---

## 2026-09-22 — Herramientas de línea de comandos: Chocolatey en vez de winget

**Qué se decidió:** Instalar la Databricks CLI vía Chocolatey.

**Por qué:** `winget` no estaba disponible en el sistema (no viene preinstalado en todas las versiones de Windows). Chocolatey es una alternativa equivalente para gestión de paquetes en Windows.

**Problema encontrado y resuelto:** La primera instalación de Chocolatey había quedado incompleta (la carpeta `C:\ProgramData\chocolatey` existía pero sin el ejecutable `choco.exe`). Se resolvió eliminando la carpeta corrupta y reinstalando desde cero.

---

## 2026-09-22 — Autenticación: OAuth vía perfil de CLI

**Qué se decidió:** Autenticar la Databricks CLI con `databricks auth login --host <workspace-url>`, usando un perfil nombrado `steam-project` (en vez de dejarlo como `DEFAULT`).

**Por qué:** OAuth es el método moderno recomendado — evita generar y copiar tokens manualmente. Nombrar el perfil explícitamente evita confusión si en el futuro se autentican otros workspaces (como el del proyecto de equipo SICOP, que usa credenciales separadas).

---

## 2026-09-22 — Gestión de secretos: secret scope de Databricks

**Qué se decidió:** Crear un secret scope (`databricks secrets create-scope steam-project`) y guardar la API key de Steam ahí (`api-key`), leída en notebooks vía `dbutils.secrets.get()`.

**Por qué:** Nunca hardcodear credenciales en código o notebooks. Los secret scopes son el mecanismo estándar de Databricks para esto — cifrado, y el valor nunca aparece expuesto en logs ni en el historial de comandos.

**Nota aclaratoria:** Esto es distinto de los secrets configurados en una Databricks App (que se declaran vía `app.yaml`) — mismo motor de fondo, pero este proyecto solo usa notebooks, no Apps.

---

## 2026-09-23 — Descartar `IStoreService/GetAppList` como tercera fuente

**Qué se decidió:** No usar el endpoint `IStoreService/GetAppList` (que requiere API key) para obtener la lista de appids candidatos. Usar solo Steam Store API + SteamSpy.

**Por qué:** Ese endpoint solo devuelve una lista pelada de appids, sin género ni fecha de lanzamiento — de todas formas iba a ser necesario consultar la Store API para esos datos. Elimina una dependencia y la fricción de manejar una API key adicional en el pipeline de ingesta.

**Alternativa evaluada:** Usar `SteamSpy request=all&page=N` para obtener la lista de candidatos. Se descartó como fuente principal porque viene ordenado por cantidad de dueños (no por fecha ni género) y tiene un límite de 1 solicitud por minuto — impráctico para encontrar indies pequeños de 2023-2024 sin paginar excesivamente.

---

## 2026-09-23 — Forzar idioma inglés en la Store API

**Qué se decidió:** Agregar `&l=english` a las URLs de la Store API.

**Por qué:** Sin ese parámetro, la API devuelve el campo `genres` localizado según el idioma por defecto del request (se observó `"Estrategia"` en vez de `"Strategy"`). El filtro de género para el proyecto busca el string `"Indie"` — forzar inglés evita que el filtro se rompa silenciosamente si el idioma por defecto cambia.

---

## 2026-09-23 — Endpoint de género de SteamSpy: pendiente de resolver

**Qué se encontró:** `steamspy.com/api.php?request=genre&genre=Indie` devuelve `{}` vacío, a pesar de que la sintaxis coincide exactamente con la documentación oficial de SteamSpy.

**Estado:** Sin resolver — pendiente de reintentar desde el notebook de Databricks (no solo navegador) para descartar cacheo local, y considerar si el endpoint está temporalmente caído (dado que SteamSpy es mantenido por una sola persona, sin garantías de estabilidad).

**Plan B si sigue fallando:** Usar el campo `genres` de la Store API (fuente oficial) como criterio de filtrado de indies, y usar SteamSpy solo para `appdetails` puntual por appid (endpoint ya confirmado funcional).

---
## 2026-10-03 — Bronze crudo: raw_json en vez de columnas estructuradas

**Qué se decidió:** Las tablas bronze (`bronze_steamspy_indie`,
`bronze_store_appdetails`) guardan el JSON completo como texto en una
columna `raw_json`, en vez de parsear cada campo a su propia columna.

**Por qué:** Blinda la ingesta ante cambios de esquema en las APIs fuente —
si SteamSpy agrega, renombra o quita un campo, no rompe la ingesta ni
pierde datos silenciosamente. Es especialmente relevante dado que SteamSpy
ya mostró ser inestable (ver entrada del 2026-09-23). El parseo a columnas
estructuradas se pospone a `02_silver_cleaning`, donde se controla
explícitamente qué hacer si falta un campo.

**Alternativa descartada:** Columnas individuales por campo — más cómodo
para consultar con SQL directo, pero fràgil ante inconsistencias de
esquema entre registros.

---

## 2026-10-03 — Bug de idempotencia: `overwrite` antes de `merge`

**Qué se encontró:** La celda de escritura a Delta hacía `mode("overwrite")`
inmediatamente antes de un `MERGE` — el overwrite borraba la tabla entera
antes de que el merge pudiera fusionar nada, anulando la idempotencia
buscada.

**Por qué pasó:** El patrón de "crear tabla" y "actualizar tabla existente"
se escribieron como dos pasos secuenciales en vez de condicionales.

**Solución:** `overwrite` solo se ejecuta si `spark.catalog.tableExists(...)`
devuelve `False` (primera corrida); cualquier corrida posterior usa
`MERGE` exclusivamente.

**Validación real:** Confirmado el 2026-10-07 — de 1474 appids consultados
en total entre dos corridas distintas (49 + 1425), la tabla final quedó
con 1473 registros únicos, evidencia de que un appid repetido entre
corridas se actualizó en vez de duplicarse.

---

## 2026-10-03 — Estrategia de muestreo: corte temprano en vez de filtrado exhaustivo

**Qué se decidió:** En vez de consultar la Store API para los 61,504
appids indie y filtrar después, se samplea aleatoriamente (sin reemplazo)
y se corta el loop en cuanto se junta el tamaño de muestra objetivo
(350) de juegos dentro de la ventana de fechas.

**Por qué:** Consultar los 61,504 completos tomaría más de 8 horas dado
el límite de ~200 solicitudes/5min de la Store API. Con una tasa de
acierto observada de ~24-31%, el corte temprano reduce la ingesta a
~1,400-1,500 solicitudes (~30-35 min) en vez de decenas de miles.

**Resultado real:** 350 válidos de 1,473 consultados (23.8% de tasa de
acierto real).

