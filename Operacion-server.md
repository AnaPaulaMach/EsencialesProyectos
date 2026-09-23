# Operación de un server propio (y compartido)

**Alcance:** un server que administra el equipo, con Docker, compartido con otro equipo. La
IA sin acceso al server genera el paso a paso y la persona lo corre (`Trabajar-con-IA.md` §3).

---

## 1. En un server compartido, "lo nuestro" se define por label, no por nombre

- Lo del proyecto se reconoce por el label de compose (`com.docker.compose.project=<proyecto>`),
  no por un prefijo de nombre: los contenedores de `compose run` se llaman distinto
  (`<proyecto>-api-run-<hash>`) y un prefijo los pierde.
- Lo ajeno (contenedores, carpetas, credenciales) no se toca: si se ve algo riesgoso, se le
  avisa a su dueño.

## 2. Limpiar Docker: nunca `prune` global

`docker image prune`, `volume prune` y `system prune` barren **también lo del otro equipo**.
Se borra uno por uno, después de tres chequeos:

```bash
docker image inspect <id> --format '{{.RepoTags}} {{index .Config.Labels "com.docker.compose.project"}}'
# 1) RepoTags = []   2) label = <proyecto>
docker ps -a --filter ancestor=<id>   # 3) vacío: ningún contenedor la usa
docker rmi <id>
```

- Las imágenes no guardan datos; **los volúmenes sí** (la base). Un volumen se borra solo si
  además se sabe qué tenía.
- Cada `compose build` deja la imagen anterior huérfana (`<none>`). Con un gatillo de disco
  (p. ej. > 70 %), sumar al runbook de deploy el `rmi` del ID anterior.
- La build cache es compartida: se limpia **acordado** con el otro equipo.

💡 Un `prune` no distingue de quién es cada cosa: borra imágenes ajenas y, peor, volúmenes
parados con datos.

## 3. La salud se ve sin tener que acordarse de mirarla

- **Cartel al entrar por SSH:** una línea por servicio (arriba / caído / reiniciando), RAM,
  disco, GPU. Tiene que tardar décimas de segundo o se desactiva.
- **Parte diario** por cron, leído al entrar o con **un solo comando** (`salud`).
- Aprender qué **no** es una caída: una tarea que terminó con código 0 (un `migrate` con
  `restart: "no"`) sale `Exited (0)` y es lo esperado. Si el cartel la marca en rojo, se deja
  de leer.
- El server suele estar en **UTC**: el cron se escribe en UTC, pero las horas del reporte se
  muestran en la hora local, explícita.
- El paso siguiente (avisos push) se agrega cuando haya quién los atienda. Ojo: GitHub no te
  notifica tus propias acciones, así que un aviso por issue necesita una cuenta bot aparte.

💡 Si nadie mira el server salvo que se acuerde, el primero en enterarse de una caída es un
usuario. Un radar que no suena donde el equipo mira no existe (ver `SeguridadExp.md` §8).

## 4. La RAM del guest es el cuello, y Docker miente sobre ella

- Presupuesto de RAM **medido por servicio**, escrito. Pasarse no siempre da error: puede
  colgar el daemon de Docker.
- Dentro de un contenedor, `/proc/meminfo` puede mostrar la memoria del **host físico**, no la
  de la VM. Medir desde el guest (`free -h`, `docker stats`).
- Si hay que matar algo, por PID y solo lo propio.

## 5. Recrear un contenedor borra sus logs; reiniciarlo no

`docker compose restart` conserva los logs; `up -d` que recrea el contenedor los pierde. Lo que
se necesite para diagnosticar después (la consulta, el payload recibido) va a una tabla o a un
archivo montado, no solo al log del contenedor.

💡 El día que se necesitan los logs viejos para un análisis, un deploy que recreó el contenedor
ya se los llevó.

## 6. Candado también en los servicios internos, encendido por observación

Un servicio "interno" (el modelo, la base) que otro equipo consume por la red también lleva
API key. Se enciende en tres pasos, nunca de golpe:

1. **Modo observación:** el servicio acepta todo pero registra quién llega sin key.
2. Pedirle la key a cada consumidor que aparece, y esperar **0 requests sin key durante 24 h**.
3. **Estricta.** Con el rollback escrito al lado (variable vacía + recrear el servicio).

- Los chequeos de salud propios (`/ready`) también mandan la key, o el candado los tira.
- Si una key se pegó en un chat durante el rollout, se rota al final (ver `SeguridadExp.md` §1).

💡 Encender el candado de golpe tira a los consumidores que no tienen la key, incluidos los
que nadie tenía anotados (instancias de prueba, scripts viejos). El modo observación los
muestra antes de cortarlos.
