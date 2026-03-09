# Documento 09 — Ingesta de Datos Reales

**Proyecto:** Thickener Water Recovery Sentinel (TWS)
**Versión:** 1.0 (2026-03-08)
**Audiencia:** Ingeniero de datos del cliente / integrador de sistemas

---

## ¿Qué entrega este documento?

Este documento especifica exactamente qué datos necesita TWS para calibrar un modelo en sus datos de planta y generar diagnósticos en tiempo real. Es el contrato técnico entre el cliente y el equipo de implementación.

---

## 1. Variables requeridas

### 1.1 Variables obligatorias (sin estas, el modelo no corre)

| Variable TWS | Descripción física | Unidad | Frecuencia mín. | Sensor típico |
|---|---|---|---|---|
| `RakeTorque_pct` | Torque del rastrillo como % del máximo nominal | % | 5 min | Corriente de motor rastrillo |
| `Overflow_Turb_NTU` | Turbidez del overflow medida por instrumento | NTU | 5 min | Turbidímetro óptico overflow |
| `BedLevel_m` | Nivel del lecho de sedimentación | m | 5 min | Sensor ultrasónico / radar |
| `Qf_m3h` | Caudal de alimentación (pulpa total) | m³/h | 5 min | Medidor de flujo feed |
| `Qu_m3h` | Caudal de underflow (purga) | m³/h | 5 min | Medidor de flujo underflow |
| `Solids_u_pct` | Porcentaje de sólidos en underflow | % | 5 min | Densímetro o nuclear underflow |
| `Floc_gpt` | Dosis de floculante en g/tonelada seca | g/t | 5 min | Rotámetro + balance másico |
| `pH_feed` | pH de la alimentación | — | 5 min | Electrodo pH en línea |

**Mínimo absoluto para demo rápida:** `RakeTorque_pct` + `Overflow_Turb_NTU` + `BedLevel_m` + `Qf_m3h` + `Qu_m3h`. El modelo pierde precisión pero sigue siendo funcional.

### 1.2 Variables deseables (mejoran el diagnóstico — Nivel 2)

| Variable TWS | Descripción física | Unidad | Notas |
|---|---|---|---|
| `Solids_f_pct` | % sólidos en alimentación | % | Si disponible desde densímetro feed |
| `WaterRecovery_proxy` | Fracción agua recuperada (Qu/Qf balance) | — | Calculable internamente si tienes Qu + Qf |
| `BedPressure_kPa` | Presión hidrostática de cama | kPa | Si hay sensor de presión en cono |
| `T_amb_C` | Temperatura ambiente cerca del espesador | °C | Estación meteorológica de planta o DCS |
| `Qf_dilution_m3h` | Caudal de dilución si aplica | m³/h | Solo espesadores con sistema de dilución |
| `FeedDilution_On` | Señal binaria dilución activa | 0/1 | Tag digital DCS |

### 1.3 Variables de laboratorio (desbloquean diagnóstico P2/P3 — Nivel 3)

| Variable TWS | Descripción física | Unidad | Delay típico | Notas |
|---|---|---|---|---|
| `P80_lab_um` | P80 granulométrico del feed (tamiz o láser) | µm | 4–24h (turno) | Muestra por turno; valores ~20–300 µm |
| `Clay_pct_lab` | Porcentaje de arcillas en feed (XRD o Rietveld) | % peso | 12–48h (día) | Análisis mineralógico; valores ~3–15% |

> **Nota sobre delay:** Los modelos TWS están diseñados para datos con delay de laboratorio. NO es necesario sincronización perfecta — el pipeline maneja el delay automáticamente.

### 1.4 Variable de plan de mina (opcional — Nivel 3 avanzado)

| Variable TWS | Descripción física | Unidad | Notas |
|---|---|---|---|
| `Clay_advance_signal_12h` | Predicción de contenido arcilla 12h adelante | % | Solo si el plan de mina está integrado en el DCS/PI Historian |

---

## 2. Formato esperado

### 2.1 Archivo

| Parámetro | Requerimiento |
|---|---|
| Formato | CSV (preferido) o Parquet |
| Encoding | UTF-8 |
| Separador CSV | coma (`,`) |
| Timestamp | Columna `timestamp` en formato ISO 8601: `2024-01-15T08:05:00` |
| Zona horaria | UTC preferido; si es local, indicar offset explícito (ej: `UTC-4`) |
| Valores nulos | NaN, `NULL`, celda vacía (todos son aceptados) |

### 2.2 Frecuencia de muestreo

| Frecuencia | Estado |
|---|---|
| 1 minuto | ✓ Ideal (remuestreo automático a 5 min) |
| 5 minutos | ✓ **Estándar recomendado** |
| 15 minutos | ✓ Aceptable (pérdida marginal de detección temprana) |
| 30 minutos | ⚠ Funcional pero limita la ventana de alarma temprana |
| > 30 minutos | ✗ No recomendado |

### 2.3 Duración del historial

| Duración | Estado | Nota |
|---|---|---|
| < 3 meses | ✗ Insuficiente | No cubre episodios de arcilla completos |
| 3–6 meses | ⚠ Mínimo funcional | Solo válido si incluye al menos 2 episodios de arcilla |
| **6–12 meses** | ✓ **Recomendado** | Cubre estacionalidad operacional |
| > 12 meses | ✓ Óptimo | Permite validación cruzada temporal robusta |

**Requerimiento crítico de contenido:** el historial debe incluir **al menos 2 episodios de degradación operacional** (alta turbidez sostenida) para que el modelo tenga señal de entrenamiento.

---

## 3. Mapeo de nombres de tags (tag mapping)

Los tags en sistemas de control (PI Historian, OSIsoft, WinCC, Ignition) tienen nombres propios de cada planta. El pipeline TWS usa un esquema interno estandarizado. El cliente debe proveer un archivo de mapeo:

### 3.1 Archivo de mapeo (requerido)

**Nombre:** `tag_mapping.json`

```json
{
  "timestamp":          "Timestamp",
  "RakeTorque_pct":     "ESP01_TORQUE_PCT",
  "Overflow_Turb_NTU":  "ESP01_TURB_OVF_NTU",
  "BedLevel_m":         "ESP01_BED_LEVEL_M",
  "Qf_m3h":             "ESP01_FLOW_FEED_M3H",
  "Qu_m3h":             "ESP01_FLOW_UF_M3H",
  "Solids_u_pct":       "ESP01_DENS_UF_SOLIDS",
  "Floc_gpt":           "ESP01_FLOC_DOSE_GPT",
  "pH_feed":            "ESP01_PH_FEED",
  "_optional": {
    "T_amb_C":          "MET_STATION_TEMP_C",
    "P80_lab_um":       "LAB_P80_FEED_UM",
    "Clay_pct_lab":     "LAB_CLAY_PCT_FEED"
  }
}
```

> Las variables bajo `_optional` que no existan en los datos del cliente son simplemente ignoradas.

### 3.2 Alternativa: renombrar columnas en el CSV

Si no se provee `tag_mapping.json`, el cliente puede entregar el CSV con los nombres exactos de la Sección 1.

---

## 4. Validaciones automáticas del pipeline

El script `src/ingest_real_data.py` ejecuta las siguientes validaciones al recibir datos:

### 4.1 Validaciones de esquema

- [ ] Columna `timestamp` presente y parseable
- [ ] Variables obligatorias presentes (o mapeadas vía `tag_mapping.json`)
- [ ] Tipos de datos numéricos en columnas de medición

### 4.2 Validaciones de cobertura temporal

- [ ] Duración total ≥ 3 meses (warning si < 6 meses)
- [ ] Frecuencia de muestreo detectada automáticamente (mediana de diffs)
- [ ] Frecuencia ≤ 30 min (error si > 30 min)

### 4.3 Validaciones de calidad de datos

- [ ] NaN por columna < 20% del total (warning: 10–20%, error: > 20%)
- [ ] Sensores "pegados" (std = 0 por > 4 horas continuas)
- [ ] Valores fuera de rango físico razonable:
  - Torque: [0%, 120%]
  - Turbidez: [0, 5000] NTU
  - BedLevel: [0, 5] m
  - Flujos: [0, 5000] m³/h
  - pH: [3, 12]
  - Sólidos: [0%, 80%]
- [ ] Timestamps duplicados (se eliminan, se conserva el primero)
- [ ] Gaps > 4 horas (se registran en reporte de calidad)

### 4.4 Reporte de calidad de salida

El script genera `data/real/data_quality_report.json` con:
- Cobertura temporal (inicio, fin, duración en días)
- Frecuencia de muestreo detectada
- % NaN por columna
- Lista de períodos con gaps > 4h
- Lista de sensores pegados detectados
- Veredicto final: `PASS` / `WARNING` / `FAIL`

---

## 5. Manejo de datos imperfectos

| Problema | Estrategia TWS |
|---|---|
| Gaps < 30 min | Interpolación lineal |
| Gaps 30 min – 4 h | Forward fill + columna flag `data_interpolated` |
| Gaps > 4 h | Excluir segmento del entrenamiento; marcar como `NaN` en predicción |
| Sensores pegados | Excluir período del entrenamiento; imputar con NaN en predicción |
| Mantenimiento programado | Detectar por caída a 0 en Qu + Qf simultánea; excluir del entrenamiento |
| Turbidez > 5000 NTU (spike) | Reemplazar por NaN + flag `spike_detected` |
| Frecuencia irregular | Remuestrear a 5 min con resample(mean) + interpolación |

---

## 6. Proceso de implementación — línea de tiempo

```
Semana 1: Cliente entrega datos históricos (6–12 meses)
  → TWS ejecuta src/ingest_real_data.py
  → Se genera data_quality_report.json
  → Reunión de revisión: gaps, sensores problemáticos, cobertura de eventos

Semana 2: Feature engineering sobre datos reales
  → 02_feature_engineering adaptado a columnas disponibles
  → Validación de distribuciones vs dataset sintético (sanity check)

Semana 3: Entrenamiento del modelo en datos reales
  → 03_modeling con TimeSeriesSplit sobre datos reales
  → 04_diagnosis si lab data disponible
  → Reporte de performance (F1-macro esperado ≥ 0.50 en datos ruidosos reales)

Semana 4: Validación operacional
  → Demo en dashboard (src/app.py) con datos históricos
  → Revisión con operadores: ¿los perfiles son reconocibles?
  → Ajuste de thresholds si hay feedback operacional
```

---

## 7. Contrato de datos — resumen ejecutivo (una página)

### Lo que entrega el cliente

1. **Archivo de datos**: CSV o Parquet, 6+ meses de historial, frecuencia 5 min
2. **Columnas mínimas**: las 8 variables obligatorias (Sección 1.1) bajo cualquier nombre
3. **Archivo de mapeo**: `tag_mapping.json` con correspondencia de nombres de tags
4. **Contexto operacional**: fechas aproximadas de eventos conocidos (paradas, cambios de mineral, problemas de arcilla) para validación

### Lo que entrega TWS

1. **Modelo calibrado**: Random Forest P0-P6 entrenado en sus datos
2. **Dashboard operacional**: src/app.py configurado para sus tags
3. **Reporte de calidad de datos**: análisis de cobertura, gaps, sensores problemáticos
4. **Reporte de performance**: F1-macro por perfil, matriz de confusión, SHAP top features
5. **Manual de operación**: qué significan los perfiles en el contexto de su planta

### Plazo estimado

4 semanas desde recepción de datos hasta entrega del modelo calibrado.

### Qué NO cubre este contrato

- Instalación de nuevos sensores o modificación del DCS
- Integración en tiempo real con el DCS/Historian del cliente (requiere acuerdo adicional)
- Etiquetado manual de eventos históricos (el cliente puede acelerar esto con contexto operacional)

---

## 8. Preguntas frecuentes

**¿Qué pasa si no tenemos datos de laboratorio (P80, Clay_pct)?**
El modelo corre con DCS-only (Nivel 1). Se pierde el diagnóstico preciso P2 vs P3, pero el clasificador general P0-P6 funciona. En la práctica, la acción operacional para P2/P3 es similar hasta que llegue el lab.

**¿Necesitamos etiquetas de eventos en el historial?**
No. TWS genera las etiquetas automáticamente basadas en los patrones físicos del modelo. El cliente puede proveer fechas de eventos conocidos opcionalmente para validación.

**¿Qué frecuencia mínima funciona?**
15 minutos es el límite funcional. A 30 min el modelo pierde la ventana de alerta temprana (30 min = 1 solo punto de anticipación).

**¿Funciona con datos de múltiples espesadores?**
Actualmente el modelo es por-espesador. Si la planta tiene varios espesadores con geometría similar, el modelo entrenado en uno puede ser punto de partida para los demás (transfer learning — conversación aparte).

**¿Qué pasa si el historial no tiene eventos de arcilla?**
El modelo no tendrá señal para perfiles P2/P3. Funcionará para P0/P5/P6. En ese caso se recomienda extender el historial hasta capturar al menos un episodio de alta arcilla.

---

*Documento 09 — Ingesta de Datos Reales | TWS v13.0b | 2026-03-08*
