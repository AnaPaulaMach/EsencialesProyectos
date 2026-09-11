# Método: cómo se organiza un proyecto

> **Para quien lee (persona o IA):** reglas que ya pagaron su costo. Cada una lleva 💡 la lección que la originó. Si sos una IA y te mandaron acá: recorré las reglas, decí cuáles faltan en el repo donde estás trabajando y no des por hecho que algo se cumple sin verificarlo.

## 1. Un solo lugar para "qué hacer ahora": el tablero

- GitHub **Issues + Project**, con *auto-add* de todo issue abierto a la columna **Bandeja**.
- La Bandeja es la **única puerta de entrada**: pedidos, bugs, ideas, hallazgos de CI. Nada
  se trabaja si no tiene issue.
- **Triage semanal** (día fijo): la Bandeja se clasifica, se prioriza o se descarta. Lo que
  lleva semanas sin moverse se marca *stale* y se decide.
- Límite de trabajo en curso (WIP) por persona: si está lleno, no se arranca otra cosa.

💡 Sin esto, las tareas viven en chats, memorias y cabezas; a la semana nadie sabe qué está
abierto. El tablero también es a donde deben llegar los avisos automáticos (ver Seguridad §8).

## 2. Cada documento tiene UN rol

| Doc | Rol |
|---|---|
| Tablero | qué hacer ahora, estado, prioridad |
| `decisiones.md` | decisiones formales numeradas (**D-XX**) — fuente de verdad |
| `MAPA.md` | técnica ↔ archivo ↔ símbolo (para encontrar código) |
| `metas_y_requerimientos.md` | el *por qué* del proyecto, conceptual |
| `ERRORES_USO_REAL.md` | bugs vistos con usuarios reales |
| `HISTORICO.md` | lo cerrado o superado — **no es estado vigente** |
| `docs/` | runbooks (deploy, salud) + hallazgos puntuales + contratos con otros equipos |

💡 Un doc con dos roles se vuelve un doc que nadie actualiza. `HISTORICO.md` existe para que
el estado vigente no se llene de pasado; **buscar ahí antes de asumir que algo no se hizo**.

## 3. Decisiones numeradas

- Toda decisión que cierre un debate va a `decisiones.md` como **D-XX**: qué se decidió, por
  qué, qué se descartó. Se referencia por número en issues, commits y código.
- Se sincroniza con el repo cada tanto (un skill `/sync-decisiones` lo hace en automático).

💡 "¿Por qué hicimos esto así?" se pregunta seis meses después. Sin D-XX, la respuesta es
re-debatir.

## 4. Comentarios en código: máx 2 líneas

`# #138: el campo expediente SIEMPRE gana sobre el nº del texto`
Número de issue + **el invariante**. La historia, el debate y el detalle van al issue.
Un radar en CI (`medir_comentarios.py`) falla si un bloque se pasa.

💡 Los comentarios largos envejecen mal y nadie los lee; el issue tiene fecha, autor y contexto.

## 5. Funciones bajo un tope

Radar en CI (`medir_funciones.py`): líneas y anidamiento máximos. Si una función se pasa,
se **parte en etapas nombradas**, no se sube el tope. Excepciones en una lista explícita, con
el issue que las justifica.

💡 Una función de 380 líneas es la que nadie quiere tocar. El tope hace el problema visible
el día que nace, no el día que hay que arreglarla.

## 6. Ante un fallo: medir la CLASE, no arreglar el caso

1. ¿De qué **clase** es este error? (no "esta pregunta", sino "preguntas por X con Y")
2. ¿Qué **tasa** tiene la clase? Medirla con un script sobre datos reales (el *radar*).
3. Si es frecuente → **UNA palanca general**. Si es rara → anotar y seguir.

💡 Los fixes caso por caso se acumulan como parches que nadie entiende. Y medir el agregado
esconde clases: una clase al 0,3% puede ser la que afecta a 4 de 5 usuarios reales.

## 7. Evidencia versionada, sin basura

- Todo output de una medición (JSON testigo, CSV, reporte) va **bajo el repo**, en una
  carpeta de evaluación, nunca en el home del server.
- Lo que se valida se archiva como resumen en un `.md`; el CSV crudo no se versiona.
- Lo que el server escribe en cada corrida (históricos) **no** se versiona: choca en cada pull.

💡 Un `.gitignore` mal pensado o un archivo trackeado "por accidente" te da conflictos en
cada deploy.

## 8. Fechas absolutas, siempre

En docs, issues, memoria y commits: `2026-09-11`, nunca "hoy", "ayer", "la semana pasada".

💡 "Hoy" en un documento es una fecha que nadie puede recuperar.

## 9. Commits

Título corto en español + cuerpo que explica el **por qué**, con el número de issue.
Los commits y pushes los hace una persona (ver `Trabajar-con-IA.md`).
