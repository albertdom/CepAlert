# CepAlert

App Android que predice zonas con probabilidad de **Boletus edulis** (cep) en los Pirineos catalanes, combinando el tipo de bosque con datos meteorológicos y de suelo. Toda la predicción se calcula **en el dispositivo**, sin backend.

> Proyecto personal. MVP funcional; aún no publicado en Google Play.

## Qué hace

- Mapa con las zonas forestales puntuadas por su probabilidad de cep, sobre MapLibre (OpenStreetMap).
- **Scoring por zona** a partir de variables reales, con desglose de por qué una zona puntúa alto o bajo.
- Overlay WMS de coberturas del suelo (ICGC) y filtro de zonas.
- Datos meteorológicos actualizados a diario mediante un workflow de GitHub Actions.
- Caché en disco de 6 h para funcionar rápido y con poca red.

## Cómo calcula el score

El score de cada zona es una combinación ponderada de factores normalizados:

```
score = humedad × 0,4 + lluvia(10 días) × 0,3 + bosque_compatible × 0,2 + temperatura × 0,1
```

El factor de bosque sale de la clasificación de especies de cada zona (enriquecida con tipo de bosque y pH de suelo); el resto, de la meteorología reciente.

## Stack

- **Kotlin** · **Jetpack Compose** · **Material 3**
- **MapLibre GL** (mapas, gratis, sin API key de pago)
- **Retrofit** + **KotlinX Serialization** (API meteorológica)
- **Hilt** (inyección de dependencias) · **Coroutines**
- Sin backend: el scoring corre en cliente sobre un GeoJSON empaquetado en la app.

## Datos

- **Zonas forestales:** generadas a partir de MFE50 (Mapa Forestal de España) y modelo digital del terreno, enriquecidas con tipo de bosque y pH de suelo (SoilGrids / HIC). El pipeline está en Python en [`tools/build_geojson/`](tools/build_geojson/).
- **Meteorología:** Open-Meteo + AEMET, refrescada a diario por GitHub Actions y empaquetada como asset de la app.

## Estructura

- `app/` — la app Android (UI Compose, motor de scoring, repositorios, mapa).
- `tools/build_geojson/` — pipeline de datos que produce `forest_zones.geojson` (ver su propio README).
- `docs/` — spec y plan de implementación del MVP.

## Compilar

Requiere JDK 17 y el SDK de Android.

```bash
./gradlew assembleDebug        # APK de pruebas
./gradlew testDebugUnitTest    # tests (motor de scoring, parsers, caché)
```

El GeoJSON de zonas se genera aparte con el pipeline de `tools/build_geojson/` (requiere descargar MFE50 y el modelo del terreno; ver sus instrucciones).

## Estado

MVP funcional: mapa, scoring completo, overlay de coberturas, filtro de zonas y pipeline meteorológico diario. Pendiente: pasos de publicación en Play Store (icono, firma de release, ficha).
