# Metodología — Definición de la muestra

Este documento registra las decisiones de diseño detrás de la muestra de datos usada en el análisis de supervivencia de juegos indie, y el razonamiento detrás de cada una. Estas decisiones son punto de partida para v0.1, no reglas fijas — se documentan los criterios para que cualquier cambio futuro sea trazable.

## 1. Criterio de "juego indie"

**Decisión:** género `"Indie"` presente en el campo `genres` de la respuesta de Steam Store API (`appdetails`).

**Por qué:** es el criterio más simple y disponible directamente en la fuente principal de datos, sin depender de una segunda fuente (SteamSpy) para la clasificación. Facilita reproducibilidad: cualquiera que corra el pipeline con la misma API obtiene la misma muestra.

**Limitación conocida:** el género es autodeclarado por el desarrollador/publisher al subir el juego a Steam — no hay validación externa de qué es "verdaderamente" indie (ej. un estudio con financiamiento grande puede etiquetarse igual que un desarrollador solo). Esto puede introducir ruido en la muestra.

**Posible ajuste futuro:** cruzar con SteamSpy para excluir indies con `owners` o `positive` reviews por encima de cierto umbral (indies que ya "explotaron" y se comportan más como AAA).

## 2. Ventana de fecha de lanzamiento

**Decisión:** `release_date` entre **enero 2023 y diciembre 2024**.

**Por qué:** ninguna de las dos APIs (Store API, SteamSpy) ofrece datos históricos — ambas devuelven únicamente el estado actual del juego. Para medir "supervivencia al año 1" sin series de tiempo reales, la estrategia es usar juegos cuyo primer año **ya haya transcurrido**, y tratar su estado actual (CCU, reviews vía SteamSpy) como proxy del resultado. Esta ventana garantiza que, al momento del análisis, haya pasado al menos un año completo desde el lanzamiento.

**Limitación conocida:** es un diseño *cross-sectional* (una foto del presente), no una serie de tiempo real — no se puede ver la trayectoria mes a mes del juego, solo su punto de partida (metadata de lanzamiento) y su estado actual.

**Posible ajuste futuro:** angostar o mover la ventana según qué tan grande resulte la muestra real una vez aplicado el filtro; o iniciar una captura de snapshots diarios/semanales a partir de ahora para construir series de tiempo propias de cara a versiones futuras del proyecto.

## 3. Tamaño de la muestra

**Decisión:** 300–400 juegos.

**Por qué:** balance entre tener suficiente volumen para que el análisis tenga peso estadístico, y mantenerse dentro de un tiempo de ingesta razonable dado el límite de ~200 solicitudes cada 5 minutos de Store API.

**Limitación conocida:** no es necesariamente representativo de *todos* los indies lanzados en el período — es una muestra, no el universo completo.

**Posible ajuste futuro:** aumentar el tamaño de muestra en una versión posterior si el análisis inicial muestra señales interesantes que valga la pena validar con más volumen.