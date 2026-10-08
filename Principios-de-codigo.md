# Principios de código: lo que hay que sacar afuera, medir y dibujar

**Alcance:** reglas de diseño que aplican a cualquier repo con pipeline (RAG, chatbot, ingesta).
Fuente principal: *The Pragmatic Programmer*, 20th Anniversary Edition (Thomas y Hunt). Notas de
lectura propias en [Libros/Tech & Code](https://github.com/AnaPaulaMach/Libros/blob/main/Tech%20%26%20Code/The%20Pragmatic%20Programmer%20-%20Reading%20Notes.md).
Anotado 2026-10-08.

---

## Checklist rápida

- [ ] Todo lo que se sabe que va a cambiar está fuera del código (`.env`, YAML, tabla), detrás de **un** módulo
- [ ] Ninguna ventana rota sin issue: `try/except` que traga, prompt parchado, `TODO` sin número
- [ ] Los parámetros clave (chunk, umbrales, top-k, modelo) tienen un comentario con el **por qué**
- [ ] Cambiar de modelo, de vector store o de canal no obliga a tocar más de una capa
- [ ] Lo que se asume, se prueba: eval chico con respuesta esperada antes de optimizar nada
- [ ] Cada etapa del pipeline valida entrada y salida (pydantic); los invariantes no se asumen
- [ ] Antes de "optimizar": 3 o 4 tamaños de entrada, un gráfico
- [ ] Antes de paralelizar: un diagrama de qué pasos son independientes y dónde se sincronizan
- [ ] Si es un chatbot o agente: máquina de estados dibujada, con los estados de error
- [ ] Auditoría de seguridad automática, semanal y al salir cada modelo nuevo

---

# Diseño

## 1. Configuración externa, detrás de una API fina

**Qué:** parámetros por entorno, valores impuestos desde afuera, rutas, prompts, claves, modelo y
proveedor, tamaño de chunk, top-k, umbrales. Todo a un "balde" de configuración; cambiar un valor
no debe requerir recompilar ni redeploy de código.

**Cómo:** un solo módulo (`config.py`, idealmente `pydantic-settings`) que todo el resto importa.
Nadie más lee `os.environ`. `.env.example` documenta cada variable con su default.

💡 Código sin configuración externa es "código dodo": no se adapta y se extingue. La API fina
desacopla el código de cómo está representada la config (archivo hoy, servicio mañana).
Configuración como servicio (dinámica, con notificaciones de cambio) existe, pero no hace falta
para un equipo chico con una instancia. (Cap. 5, "Configuration".)

## 2. ETC: ¿lo que acabo de hacer deja el sistema más fácil o más difícil de cambiar?

- **Ortogonalidad:** cambiar una cosa no afecta otra. En un RAG las capas son ingesta,
  indexado, recuperación y generación. Si cambiar de vector store o de modelo toca varias, falta
  ortogonalidad.
- **Reversibilidad:** interfaz fina sobre el cliente del LLM y sobre la base. Probar otro modelo
  tiene que ser una tarde, no una semana. Los modelos cambian cada pocos meses: esta es la única
  predicción de futuro que vale la pena hacer.
- **Herencia es acoplamiento.** Preferir protocolos, delegación, composición.
- **DRY:** lo repetido queda desactualizado. Incluye docs y prompts.

(Cap. 2 y 5.)

## 3. Tracer code, no prototipo perfecto

Primero un camino fino y completo de punta a punta (PDF → chunk → embed → pregunta → respuesta,
con 10 documentos). Después se engorda cada etapa. Lo contrario, perfeccionar el OCR antes de
tener nada funcionando, es lo que deja proyectos de meses sin nada desplegado. El prototipo es
código descartable para despejar un riesgo; el tracer queda. (Cap. 2.)

## 4. Diseño por contrato

Por cada etapa: qué entra, qué sale, qué nunca puede pasar ("un chunk siempre tiene
`expediente_id`"). Se valida con pydantic en el borde, no se asume. El manejo de errores no debe
eclipsar la lógica: validar en el borde y confiar adentro. (Cap. 4.)

## 5. Ventanas rotas

Un `except: pass` en el OCR, un prompt con un parche "temporal", un chunking que "más o menos
anda". Se arregla o se anota con fecha en el tablero el mismo día. Lo que no se puede arreglar se
anota igual (ver `Metodo.md` §1). (Cap. 1.)

## 6. Comentarios: el por qué, nunca el cómo

El código muestra cómo. El comentario dice por qué el chunk es de 800 tokens o por qué se filtra
ese metadato, con número de issue (formato en `Metodo.md` §4). Una decisión sobre un LLM sin
justificación escrita se olvida en dos semanas. (Cap. 1.)

## 7. Involucrar al usuario en el trade-off

¿Respuesta rápida con 90% de precisión, o lenta con revisión humana? Eso define la arquitectura.
Se pregunta antes de diseñar, no se adivina. Ningún software es perfecto: que sea usable. (Cap. 1.)

---

# Medir

## 8. Testear lo que asumimos, no solo el código

En un pipeline con IA los supuestos no probados son el riesgo principal: "el retriever trae el
expediente correcto", "el OCR lee tablas", "el modelo no inventa artículos". Se prueban con un
eval chico (20-30 preguntas con respuesta esperada) que corre como radar (`Metodo.md` §10-§14).
(Cap. 7.)

## 9. Big-O: medir antes de optimizar

- O(n²) significa que duplicar la entrada cuadruplica el tiempo. Es cota superior, "del orden de".
- Un bucle O(n²) simple le gana a un O(n log n) complejo con n chico.
- Con pocos datos todo parece lineal; con millones el sistema pagina y se desploma. Un sort
  probado con claves al azar sorprende con entrada ordenada.
- **Cómo:** correr con 3 o 4 tamaños de entrada y graficar. La forma de la curva (sube, recta,
  se aplana) dice más que la teoría. Media hora que evita optimizar lo que no es el cuello.

Casos: antes de pedir más GPU, medir latencia con 1/2/4/8 usuarios. Antes de cambiar el modo de
retrieval, medir con 100/1k/10k/100k chunks. Antes de ingestar un expediente de 2.000 páginas,
tiempo de OCR contra páginas. (Cap. 7, "Algorithm Speed".)

## 10. Concurrencia: dibujar antes de paralelizar

Pasos escritos en serie muchas veces son independientes. Un diagrama de actividad (cajas por
paso, barras gruesas donde se sincroniza) muestra qué puede correr en paralelo y dónde está el
punto de encuentro. En una ingesta: OCR, embeddings y metadatos de distintos documentos son
independientes; la escritura al índice sincroniza.

💡 Sirve tanto para acelerar como para decidir qué **no** paralelizar: dos etapas que pelean por
la misma GPU o la misma RAM en paralelo van más lento que en serie. Dibujar antes de subir
`MAX_PARALELAS`, no después de que se cuelgue el daemon. (Cap. 6.)

## 11. Un chatbot es una máquina de estados

Un agente con herramientas es un bucle de eventos: mensaje, llamada, resultado, decisión. Dibujar
los estados (esperando entrada, recuperando, generando, pidiendo aclaración, error, timeout,
derivado a humano) encuentra los que hoy no se manejan. Los bugs "el bot queda en bucle" son
siempre transiciones que faltan. (Cap. 5, "Finite State Machines".)

---

# Seguridad

## 12. Auditoría automática cada vez que sale un modelo nuevo

**Qué:** un agente que audita el repo una vez por semana y además a mano cuando sale un frontier
model nuevo.

💡 Un modelo más capaz encuentra vulnerabilidades que el anterior no veía, y ese mismo modelo lo
tiene el atacante. El software no cambió; subió el nivel de quien lo mira. Quienes lo tienen
automatizado reportan que encuentra cosas que daban por bien hechas.

**Cómo (verificado 2026-10-08 contra la doc de Claude Code):**
- Primera corrida a mano con `/security-review` o pidiendo auditoría completa. Pedirle lo
  específico del proyecto: inyección vía documento indexado, fuga entre expedientes, extracción
  del prompt de sistema, secretos en logs, dependencias con CVEs.
- Automatizar con una **routine en la nube** de Claude Code (cron, corre con la suscripción, sin
  API key, no necesita la PC prendida). Alternativa: GitHub Actions con `claude-code-action` y
  `claude setup-token`, que también cobra a la suscripción, pero verificar los términos antes
  de dejarlo fijo.
- `/loop` y tareas de sesión no sirven: mueren con la sesión.
- Si hay `ANTHROPIC_API_KEY` en el entorno, `claude -p` la usa antes que el login y cobra por API.

Se suma a la checklist de `SeguridadExp.md` (betterleaks, pip-audit, Dependabot): esas miran
secretos y CVEs conocidas; esta mira la lógica.

Fuentes: hilo en X de @marcoporracin (2026-10-07) y respuestas;
[routines](https://code.claude.com/docs/en/routines), [github-actions](https://code.claude.com/docs/en/github-actions),
[authentication](https://code.claude.com/docs/en/authentication).
