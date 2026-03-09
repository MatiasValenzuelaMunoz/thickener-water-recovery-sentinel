# Thickener Water Recovery Sentinel (TWS)

**Detección temprana y diagnóstico de crisis operacionales en espesadores Cu/Mo**

> *"El sensor de turbidez alarma cuando ya es tarde. TWS alarma antes — y te dice por qué."*

---

## El problema

Un espesador convencional Cu/Mo corriendo por debajo del benchmark de recuperación de agua pierde dinero en silencio. En una operación de escala media (~1.75 Mt/año):

| Fuente de pérdida | Estimación anual |
|---|---|
| Gap de recuperación de agua (79% vs benchmark 90%) | USD 4,190,000 |
| Exceso de floculante por inconsistencia entre turnos | USD 62,000 |
| Paradas no planificadas (54 h/año) | USD 65,000 |
| **Total recuperable** | **~USD 4,300,000 / año** |

La raíz del problema es invisible sin contexto mineralógico: la **arcilla sericita** (muscovita, P80 20–50 µm) colapsa la velocidad de sedimentación. El operador ve turbidez alta y aumenta el floculante — lo cual empeora la situación (paradoja del bogging). El DCS estándar no puede distinguir sericita de kaolinita.

---

## La solución

TWS implementa tres niveles de detección, en orden creciente de información requerida:

```
Nivel 0 — CUSUM sobre torque
  Detecta cambios de régimen con un solo tag DCS.
  Output: "esto no es normal"

Nivel 1 — Clasificador de perfiles (DCS-only)
  Identifica 7 perfiles operacionales con los tags DCS estándar.
  Sin hardware adicional. Sin datos de laboratorio.
  Output: Normal / Arcilla / Bogging / Degradación WR / ...

Nivel 3 — Diagnóstico kaolinita vs sericita
  Distingue el tipo de arcilla con datos de laboratorio y plan de mina.
  Output: protocolo específico por tipo de arcilla
```

### Los 7 perfiles operacionales

| Perfil | Descripción | Acción |
|---|---|---|
| P0 | Normal | Monitoreo rutinario |
| P1 | Sobrecarga hidráulica | Reducir Qf, revisar dilución |
| P2 | Fouling kaolinita | Ajuste gradual floculante |
| **P3** | **Colapso sericita** | **NO aumentar floculante · dilución inmediata** |
| P4 | Falla de floculación | Revisar dosificación y pH |
| P5 | Bogging | Reducir floculante, purga |
| P6 | Degradación recuperación agua | Revisar balance de circuito |

---

## Por qué importa la distinción P2 / P3

Sin este diagnóstico, el operador aplica el mismo protocolo a kaolinita y sericita. En sericita, ese protocolo es contraproducente:

| | Kaolinita (P2) | Sericita (P3 — urgente) |
|---|---|---|
| P80 feed | 50–100 µm | **20–50 µm** |
| Yield Stress | 25–45 Pa | **60–80 Pa** |
| Respuesta al floculante | Ajuste gradual posible | **Aumentar = bogging** |
| Ventana de acción | Horas | **1–2 horas** |

---

## Resultados (dataset simulado, calibrado a espesador Cu/Mo 1.75 Mt/año, 180 días)

| Componente | Métrica | Valor |
|---|---|---|
| Clasificador P0–P6 (DCS-only) | F1-macro test | 0.610 |
| Clasificador P0–P6 | P5 Bogging F1 | 0.99 |
| Clasificador P0–P6 | P6 Degradación WR F1 | 0.97 |
| Diagnóstico P2/P3 | F1-macro | **0.866** |
| Diagnóstico P2/P3 | ROC-AUC | 0.970 |
| Diagnóstico P2/P3 | P3 recall @ thr=0.65 | **91.5%** |
| Diagnóstico P2/P3 | P3 precisión @ thr=0.65 | 78.1% |

**Lectura operacional del diagnóstico P3:** 9 de cada 10 eventos de sericita son detectados. Cuando el sistema dice "urgente", hay 78% de probabilidad de ser correcto — el 22% restante es kaolinita, que igual requiere atención pero sin la urgencia del protocolo anti-bogging.

---

## Línea de tiempo de implementación

```
Semana 1 — Recepción y validación de datos
  Cliente entrega CSV/Parquet del DCS histórico (6+ meses)
  TWS genera reporte de calidad: PASS / WARNING / FAIL

Semana 2 — Feature engineering
  411 features con justificación física sobre los datos de la planta

Semana 3 — Entrenamiento
  Modelo calibrado a los datos reales de la planta

Semana 4 — Validación + entrega
  Dashboard operacional + reporte de rendimiento por perfil
```

**Requisito mínimo (Nivel 1):** 8 tags DCS + 6 meses de historial. Sin nuevo hardware.

**Requisito Nivel 3:** laboratorio granulométrico (P80 + % arcilla por turno/día).

---

## Cómo colaborar

### Opción A — Piloto técnico
Tienes acceso a datos históricos de un espesador instrumentado. Calibramos el modelo a tu planta, validamos los perfiles con tu equipo, y entregamos un reporte de impacto económico. Sin costo de desarrollo.

### Opción B — Acceso a datos
No tienes tiempo para un proyecto formal pero puedes facilitar un export del historiador (PI/OSIsoft, Ignition, CSV). Firmamos NDA, procesamos los datos de forma anónima.

### Opción C — Conversación
Trabajas con espesadores o conoces a alguien que sí. Una llamada de 30 minutos para ver si hay fit.

---

## Especificación técnica de ingesta de datos

Qué variables se necesitan, en qué formato, con qué frecuencia y cómo manejar datos imperfectos:

→ [`data_requirements.md`](data_requirements.md)

---

## Contacto

**Matias Valenzuela** — Data Science & Process Optimization

[linkedin.com/in/matiasvalenzuelam](https://www.linkedin.com/in/matiasvalenzuelam/)

Si trabajas con espesadores instrumentados y tienes acceso a datos históricos de proceso, esto es una invitación abierta a colaborar en un piloto real.
