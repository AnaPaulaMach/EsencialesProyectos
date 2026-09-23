# Método: cómo se organiza un proyecto

**Alcance:** proyectos de software que duran meses, con 2-4 personas y usuarios reales. Para
un script de una tarde, alcanza con *Medir* §7 y *Decidir* §17.

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
| Registro de decisiones | decisiones formales numeradas — fuente de verdad (§3) |
| Mapa del código | técnica ↔ archivo ↔ símbolo |
| Histórico | lo cerrado o superado — **no es estado vigente** |
| `docs/` | runbooks + hallazgos puntuales + contratos con otros equipos |

💡 Un doc con dos roles es un doc que nadie actualiza. **Buscar en el histórico antes de
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

## 4. El rastro: comentarios cortos, commits con porqué, fechas absolutas

- **Comentarios en código, máx 2 líneas:** `# #138: el campo del formulario SIEMPRE gana sobre
  el valor del texto` — número de issue + **el invariante**. La historia va al issue. Un radar
  en CI falla si un bloque se pasa.
- **Commits:** título corto + cuerpo con el **por qué** y el número de issue.
- **Fechas absolutas** (`2026-01-31`, nunca "hoy" ni "la semana pasada") en docs, issues,
  memoria y commits.

💡 Los comentarios largos envejecen mal; el issue tiene fecha, autor y contexto.

## 5. Funciones bajo un tope, y la salida es partir

Radar en CI: líneas y anidamiento máximos. Si una función se pasa, se **parte en etapas
nombradas**; no se sube el tope. Excepciones en una lista explícita, con el porqué escrito.

💡 Cuando una función se pasa, la tentación es subir el tope. El tope existe para que el
problema se vea el día que nace.

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

Un promedio puede quedar igual mientras una clase se rompe. Y una clase **rara por volumen**
puede tocar a **casi todos los usuarios**: por volumen no existe, por personas es universal.

## 9. Separar tráfico de prueba del real antes de medir

Toda métrica sobre logs empieza por etiquetar o filtrar quién generó cada fila.

💡 El tráfico propio (pruebas, eval, experimentos) infla repeticiones y fabrica "bugs de
producción" que en producción no pueden pasar.

Un filtro que devuelve **0 filas** es un bug del filtro hasta que se demuestre lo contrario
(un valor grabado como `search` no matchea con `/search`).

## 10. Un radar mide el componente que atraviesa, no "el sistema"

Antes de declarar un radar, escribir **qué cuello ejercita**. Si no atraviesa el que duele,
mide otra cosa.

💡 Una sonda de disponibilidad puede dar 100% mientras los usuarios ven timeouts, si pega a
un endpoint que no toca el componente lento.

## 11. Radares con gatillo

Un radar puede **dormir** hasta una condición explícita ("cuando haya N ≥ 30", "cuando salga
el modelo nuevo"). El gatillo se anota junto al radar. Cuando llega, se **re-corre el
script**, no se decide de memoria.

Al cerrar una palanca, anotar **la fecha absoluta de re-medición** (una semana después) y la
meta ("la clase debe dar 0").

## 12. Radar y compuerta son cosas distintas

- **Compuerta:** corre antes de desplegar y bloquea si empeora.
- **Radar:** corre periódicamente y detecta deriva.

## 13. El remedio como enfermedad

Cada guarda, validación o regla automática es **una pieza más que falla**. Se mide como
cualquier otra: tasa de disparo y de falso positivo, con N. Sin radar propio, es un fallo
esperando fecha.

💡 Un filtro de palabras sin medir censura respuestas correctas (un apellido que coincide con
la lista) y nadie se entera.

## 14. Un evaluador automático necesita su propia validación

Medir la tasa de error **del medidor** antes de confiar en sus números. Y distinguir
**ausencia de dato** de **valor cero**: un 0 puede ser una columna que nunca se pobló.

Calibrar el detector **mirando a ojo los casos que marca** antes de contar. Y decir qué parte
del universo **no puede juzgar** (datos viejos sin la señal que usa).

💡 Una heurística obvia ("pocos caracteres = ilegible") suele marcar otra cosa (hojas en
blanco a propósito). Solo mirando los casos se ve qué señal separa bien.

## 15. La clase que temés no es la que tenés: leer los casos, no solo el %

Antes de diseñar la palanca, leer **todos** los casos de la clase (un `--todo` del radar), no
solo la tasa. La hipótesis del issue es una sospecha, no la clase.

💡 La falla que se imagina (el modelo *niega* un dato) puede no aparecer nunca, y la real ser
otra (lo *omite*). Una palanca contra la imaginada no arregla ningún caso.

## 16. Una guarda dispara por ausencia, es aditiva y cubre todos los caminos

- **Gatillo positivo:** "la respuesta no nombra el dato que debe nombrar", no una lista de
  frases prohibidas (la lista nunca está completa).
- **Aditiva:** agregar lo que falta adelante o al final; no reescribir la respuesta.
- **Todos los caminos:** revisar que los `return` tempranos (sin resultados, error, atajo)
  no salten la guarda.

💡 Un atajo que responde antes (por ejemplo "sin resultados") esquiva la guarda aunque el dato
esté disponible.

---

# Decidir

## 17. La medición no decide sola

La medición dice **qué se está pagando**; quién paga lo decide el equipo. Lo que no se puede
hacer es fingir que la medición dijo otra cosa: se registra la decisión con su costo.

💡 Adoptar la opción que perdió el A/B puede ser razonable por otras razones; lo que no es
razonable es no anotar los errores que trae.

## 18. Un error visible es mejor que uno invisible

Ante una entrada ambigua, **no adivinar**: se acepta toda corrección determinista y sin
pérdida; si hay que *elegir* entre candidatos, se pregunta o se rebota. Aflojar una validación
para "que ande" suele cambiar un error que el usuario ve (y reintenta) por uno que no ve.

💡 Resolver un identificador mal copiado al "más parecido" puede responder, con citas y todo,
sobre el caso de **otra persona**. Callar no es el daño; afirmar sí.

## 19. Un fix corre hacia adelante; lo viejo se corrige cuando se toca

Decidir y **escribir el alcance**: ¿se reprocesa lo histórico o solo lo nuevo? Reprocesar todo
cuesta y rompe cosas; lo viejo se corrige cuando el ciclo normal lo toca, o a mano si un caso
real vuelve a fallar. La regla para ese caso manual va en la decisión.

## 20. Un aviso que nunca aparece o que aparece siempre no sirve

Antes de agregar un aviso al usuario, medir cuántas veces saldría **en uso real**. Si hoy da 0,
queda un radar con gatillo y el diseño angosto anotado; si sale en un cuarto de las respuestas,
satura y nadie lo lee.
