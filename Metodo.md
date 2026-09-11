# Método: cómo se organiza un proyecto

> **Para quien lee (persona o IA):** reglas que ya pagaron su costo. Cada una lleva 💡 la lección que la originó. Si sos una IA y te mandaron acá: recorré las reglas, decí cuáles faltan en el repo donde estás trabajando y no des por hecho que algo se cumple sin verificarlo.

**Alcance:** escrito desde un proyecto de software que dura meses, con 2-4 personas y usuarios
reales. Para un script de una tarde, alcanza con *Medir* §7 y *Decidir* §15.

---

# Organizar

## 1. Un solo lugar para "qué hacer ahora": el tablero

- Issues + Project, con *auto-add* de todo issue abierto a una columna **Bandeja**.
- La Bandeja es la **única puerta de entrada**: pedidos, bugs, ideas, hallazgos de CI.
- **Triage semanal** (día fijo): se clasifica, prioriza o descarta. Lo que lleva semanas sin
  moverse se marca *stale* y se decide.
- Límite de trabajo en curso por persona.

💡 Sin esto, las tareas viven en chats y cabezas. El tablero es también a donde deben llegar
los avisos automáticos (ver `SeguridadExp.md` §8).

## 2. Cada documento tiene UN rol

| Doc | Rol |
|---|---|
| Tablero | qué hacer ahora, estado, prioridad |
| `decisiones.md` | decisiones formales numeradas — fuente de verdad |
| `MAPA.md` | técnica ↔ archivo ↔ símbolo |
| `metas_y_requerimientos.md` | el *por qué* del proyecto |
| `ERRORES_USO_REAL.md` | bugs vistos con usuarios reales |
| `HISTORICO.md` | lo cerrado o superado — **no es estado vigente** |
| `docs/` | runbooks + hallazgos puntuales + contratos con otros equipos |

💡 Un doc con dos roles es un doc que nadie actualiza. **Buscar en `HISTORICO.md` antes de
asumir que algo no se hizo.**

## 3. Decisiones numeradas, append-only

- Toda decisión que cierra un debate va a `decisiones.md` como **D-XX**.
- El **título es autocontenido**: la decisión completa + issue + fecha. Quien lee solo los
  títulos entiende el proyecto.
- **Append-only:** una decisión superada no se edita ni se borra; se agrega la nueva citando
  a la vieja ("reemplaza D-XX").
- Se referencian por número desde el código, los issues y los commits.

💡 "¿Por qué hicimos esto así?" se pregunta seis meses después. Sin número, la respuesta es
re-debatir. Sin append-only, se pierde por qué se cambió de idea.

## 4. Comentarios en código: máx 2 líneas

`# #138: el campo expediente SIEMPRE gana sobre el nº del texto` — número de issue + **el
invariante**. La historia va al issue. Un radar en CI falla si un bloque se pasa.

💡 Los comentarios largos envejecen mal; el issue tiene fecha, autor y contexto.

## 5. Funciones bajo un tope, y la salida es partir

Radar en CI: líneas y anidamiento máximos. Si una función se pasa, se **parte en etapas
nombradas**; no se sube el tope. Excepciones en una lista explícita, con el porqué escrito.

💡 Una función superó el tope y CI quedó en rojo tres días sin que nada lo frenara. La
tentación fue subir el tope. El tope existe para que el problema se vea el día que nace.

## 6. Evidencia versionada, sin basura

- Todo output de una medición va **bajo el repo**, en una carpeta de evaluación; nunca en el
  home del server.
- Lo validado se archiva como resumen en `.md`; el CSV crudo no se versiona.
- Lo que el server escribe en cada corrida **no** se versiona: choca en cada pull.

---

# Medir

## 7. Ante un fallo: medir la CLASE, no arreglar el caso

1. ¿De qué **clase** es? (no "esta pregunta", sino "preguntas por X con Y")
2. ¿Qué **tasa** tiene la clase? Un script sobre datos reales: el *radar*.
3. Frecuente → **una palanca general**. Rara → anotar y seguir.

💡 Los fixes caso por caso se acumulan como parches que nadie entiende y esconden si era
1 de 1.000 o 1 de 3.

## 8. Medir por clase Y por usuarios distintos, nunca solo el agregado

Un promedio puede quedar igual mientras una clase se rompe. Y una clase al **0,3% del
volumen** puede ser **4 de 5 usuarios**: por volumen no existe, por personas es universal.

💡 Se aprendió a los golpes, dos veces.

## 9. Separar tráfico de prueba del real antes de medir

Toda métrica sobre logs empieza por etiquetar o filtrar quién generó cada fila.

💡 Un 21% de "preguntas repetidas" justificaba un caché. Era el equipo testeando.

## 10. Un radar mide el componente que atraviesa, no "el sistema"

Antes de declarar un radar, escribir **qué cuello ejercita**. Si no atraviesa el que duele,
mide otra cosa.

💡 Una sonda de disponibilidad dio 100,00% en 10.080 minutos mientras los usuarios veían
timeouts: pegaba a un endpoint que no tocaba el componente lento.

## 11. Radares con gatillo

Un radar puede **dormir** hasta una condición explícita ("cuando haya N ≥ 30", "cuando salga
el modelo nuevo"). El gatillo se anota junto al radar. Cuando llega, se **re-corre el
script**, no se decide de memoria.

💡 Tres decisiones se tomaron así en vez de "a ojo".

## 12. Radar y compuerta son cosas distintas

- **Compuerta:** corre antes de desplegar y bloquea si empeora.
- **Radar:** corre periódicamente y detecta deriva.

💡 Confundirlos lleva a desplegar sin red o a bloquear por ruido.

## 13. El remedio como enfermedad

Cada guarda, validación o regla automática es **una pieza más que falla**. Se mide como
cualquier otra: tasa de disparo y de falso positivo, con N. Sin radar propio, es un fallo
esperando fecha.

💡 Un filtro de lenguaje censuró una respuesta correcta porque un apellido coincidía con una
palabra de la lista. Nadie había medido su tasa de falso positivo.

## 14. Un evaluador automático necesita su propia validación

Medir la tasa de error **del medidor** antes de confiar en sus números. Y distinguir
**ausencia de dato** de **valor cero**: un 0 puede ser una columna que nunca se pobló.

---

# Decidir

## 15. La medición no decide sola

La medición dice **qué se está pagando**; quién paga lo decide el equipo. Lo que no se puede
hacer es fingir que la medición dijo otra cosa: se registra la decisión con su costo.

💡 Un A/B lo ganó la opción vieja; el equipo adoptó la nueva igual, por razones que no eran
las del eval. Se anotó con sus 9 errores nuevos sin mitigar.

## 16. Fechas absolutas, siempre

`2026-09-11`, nunca "hoy" ni "la semana pasada". En docs, issues, memoria y commits.

## 17. Commits

Título corto + cuerpo con el **por qué** y el número de issue. Los commits y pushes los hace
una persona (ver `Trabajar-con-IA.md`).
