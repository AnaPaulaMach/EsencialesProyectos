# Trabajar con IA en un repo

**Alcance:** cómo se le da contexto a un asistente de código y qué se le permite hacer. Si
falta el `CLAUDE.md` o los hooks, proponerlos.

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

💡 **Una regla escrita se olvida; una que bloquea el comando, no.** La IA puede tener la regla
en el `CLAUDE.md` y romperla igual; el hook la frena.

## 3. Separar qué opera la IA y qué no

| La IA hace | La IA NO hace |
|---|---|
| editar archivos del repo | `git commit` / `git push` (salvo pedido explícito; push nunca) |
| docs, planes, decisiones, tablero | correr nada en el server |
| **sugerir siempre el commit** (título + cuerpo listo para pegar) | tocar servicios de otros equipos |
| generar el **paso a paso** para el server | ejecutar ese paso a paso |

Dejarlo explícito en el `CLAUDE.md`, porque la IA lo asume al revés: no ve la terminal de la
persona salvo que se le pegue la salida, y las credenciales y nombres de servicio se
**verifican en el repo** (`infra/`) antes de sugerir un comando.

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
  (ver `Metodo.md` §7).
- Cuando la IA se equivoca, la corrección va **a la memoria o al hook**, no solo al chat: si no,
  el error vuelve en la próxima sesión.
- Pedir analogía + ejemplo entrada→salida cuando el tema es nuevo; nunca en commits.
- Verificar afirmaciones de la IA sobre el repo **contra el código**: "ya tenés X" solo vale
  si mostró dónde.

## 7. Secretos y el chat

- Lo que se pega en un chat con una IA **se da por filtrado**: si se pegó un `.env`, una key o
  un token, se **rota**, aunque el chat sea privado.
- Para pedir ayuda con una config, pegar la salida con los valores tapados
  (`API_KEYS=front:***`), o que la IA pida solo los nombres de las variables.

💡 Al diagnosticar es fácil pegar un `.env` entero sin pensarlo. Rotar cuesta minutos; no
hacerlo es no saber nunca quién más la tiene.

## 8. Código con caracteres especiales: herramienta de edición, no shell

Regex con `\b`, barras o comillas se editan **con la herramienta de edición de archivos**, no con
heredoc / `python -c` desde el shell: el escape se pierde o mete bytes invisibles, y el test
pasa por el motivo equivocado.

## 9. La IA no habla por la persona

- Nada de `@menciones` en issues, PRs o comentarios salvo pedido explícito: la mención notifica y
  habla en nombre de la persona.
- Las preguntas para otra persona del equipo se le plantean a quien trabaja con la IA; ella
  decide si las hace y cómo.
- Si lo que hay que cambiar lo hizo otra persona, primero un comentario en **su** PR o issue
  (lo que se vio al probar, un ejemplo, abierto a que haya un motivo, ofreciendo hacer el
  cambio), en primera persona y sin la firma de la IA. Recién con su respuesta, a una issue propia.

💡 En un equipo chico, corregir el trabajo de otra sin avisar desgasta más que el bug.

## 10. Lo que solo sirve en tu máquina: parche aparte y hook que lo frena

Cambios de configuración que solo sirven para correr en local:
- se guardan como **parche** fuera del repo y se aplican sin commit;
- un hook `pre-commit` local rechaza cualquier commit que los incluya, aunque se haga
  `git add -A`; para un cambio real, una variable de entorno explícita, nunca `--no-verify`;
- los archivos solo locales (`.env.*.local`, temporales) van a `.git/info/exclude`, no al
  `.gitignore` compartido.

💡 Avisarlo en el `CLAUDE.md` no alcanza: un descuido propio, de otra sesión o de una corrida
programada los sube igual. Ver §2.

## 11. Los comandos que adivinan el destino, con destino explícito

En un fork o una copia de otro repo, `gh pr comment N` sin `--repo` puede resolver al repo
**padre**. Todo comando que publica (comentario, issue, PR) lleva el repo explícito, y se verifica
la URL que devuelve antes de darlo por hecho.

💡 Un comentario puede terminar en un repo que no corresponde, y no se puede despublicar del
todo: la notificación ya salió.

## 12. Corridas programadas: nadie está mirando

- Lo que pide aprobación (lanzar subagentes, permisos nuevos) falla en una corrida programada:
  se diseña para hacerse inline.
- Revisar sin cambiar de rama: `git show origin/<rama>:<ruta>`, `git diff A...B`, y
  `git merge-tree --write-tree A B` para ver conflictos entre PRs sin tocar el working copy.

## 13. Los bots que editan su comentario no avisan

Un revisor automático puede **editar** su comentario en cada corrida en vez de escribir uno nuevo,
y las notificaciones solo avisan de comentarios nuevos. Después de cada push, releer el
comentario y comparar `updated_at`.
