# Trabajar con IA en un repo

> **Para quien lee (persona o IA):** cómo se le da contexto a un asistente de código y qué
> se le permite hacer. Si sos una IA leyendo esto en un repo nuevo: estas son las reglas de
> la casa; si falta el `CLAUDE.md` o los hooks, proponelos. Origen: `expedientes-rag-linux`, 2026.

## 1. Un `CLAUDE.md` que orienta y apunta, no que explica

Contenido máximo, en este orden:
- qué es el proyecto (3 líneas) y qué **no** hace;
- **quién es quién** y qué terreno tiene cada uno;
- convenciones de trabajo (quién hace git, formato de comentarios, qué herramientas no usar);
- **dónde corre cada cosa** y a qué tiene acceso la IA;
- mapa de documentos (un rol por doc) y skills disponibles;
- restricciones duras (recursos, modelos, lo que nunca se toca).

💡 Un `CLAUDE.md` largo se convierte en un doc más que nadie mantiene. El detalle vive en los
docs; el `CLAUDE.md` dice **a cuál ir**.

## 2. Las reglas se hacen cumplir con hooks, no con prosa

Un hook `PreToolUse` sobre Bash que revisa el comando **antes** de correrlo y lo bloquea con
un mensaje si viola una regla:
- sin `jq` → usar `python -m json.tool` / `python -c`;
- sin `git commit` / `git push` salvo pedido explícito (override por variable de entorno);
- sin tocar contenedores/carpetas ajenas en un server compartido.

💡 **Una regla escrita se olvida; una que bloquea el comando, no.** Verificado: el hook frenó
un `--jq` que la IA iba a correr pese a tener la regla en el `CLAUDE.md`.

## 3. Separar qué opera la IA y qué no

| La IA hace | La IA NO hace |
|---|---|
| editar archivos del repo | `git commit` / `git push` (salvo pedido explícito; push nunca) |
| docs, planes, decisiones, tablero | correr nada en el server |
| **sugerir siempre el commit** (título + cuerpo listo para pegar) | tocar servicios de otros equipos |
| generar el **paso a paso** para el server | ejecutar ese paso a paso |

💡 La persona corre los comandos del server y hace git: así el rastro de quién cambió qué es
de una persona, y la IA no puede romper producción por un malentendido.

## 4. Skills = runbooks ejecutables

Un `/comando` por operación recurrente, en dos familias:
- **Operan sobre el repo → la IA los ejecuta:** `/que-sigue` (qué hacer ahora), `/anotar`,
  `/arrancar`, `/cerrar`, `/triage` (el tablero), `/sync-decisiones`.
- **Guías para el server → la IA genera el paso a paso, la persona lo corre:** `/deploy`,
  `/estado`, `/ingestar`, `/eval`.

💡 Un runbook en un `.md` se desactualiza; un skill que la IA ejecuta se prueba cada vez que
se usa.

## 5. Memoria persistente con formato fijo

- Un archivo por hecho, con frontmatter (`name`, `description`, `type`).
- Tipos: `user` (quién es la persona), `feedback` (cómo trabajar, con el **por qué**),
  `project` (estado que no se deduce del código), `reference` (URLs, tickets).
- Un **índice** de una línea por entrada; nunca contenido en el índice.
- No guardar lo que el repo ya registra (código, git, `CLAUDE.md`).
- Fechas absolutas. Consolidar cada tanto: borrar lo que quedó falso, fusionar duplicados.

💡 La memoria refleja lo que era cierto **cuando se escribió**: si nombra un archivo o un
flag, verificar que exista antes de recomendarlo.

## 6. Cómo pedir y cómo corregir

- Ante un fallo puntual, pedir la **tasa de la clase y una palanca general**, no el fix del caso
  (ver `Metodo.md` §6).
- Cuando la IA se equivoca, la corrección va **a la memoria o al hook**, no solo al chat: si no,
  el error vuelve en la próxima sesión.
- Pedir analogía + ejemplo entrada→salida cuando el tema es nuevo; nunca en commits.
- Verificar afirmaciones de la IA sobre el repo **contra el código**: "ya tenés X" solo vale
  si mostró dónde.

## 7. Lo que la IA no ve

Dejarlo explícito en el `CLAUDE.md`, porque la IA lo asume al revés:
- no tiene acceso al server ni a la DB: **da el paso a paso, la persona corre**;
- no ve la terminal de la persona salvo que se le pegue la salida;
- las credenciales y nombres de servicio se **verifican en `infra/`** antes de sugerir un comando.
