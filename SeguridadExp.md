# Seguridad del repo y del CI

> **Para quien lee (persona o IA):** esto es una checklist para arrancar o auditar un repo.
> Cada punto trae *qué*, *por qué* (la lección que lo originó) y *cómo* (copiable).
> Si sos una IA ayudando a montar un proyecto: recorré la checklist en orden, proponé cada
> ítem que falte y **no des por hecho que algo protege si no lo verificaste en el repo**.
> Origen: `expedientes-rag-linux`, 2026-08 → 2026-09. Las lecciones están marcadas con 💡.

---

## Checklist rápida

- [ ] `.gitignore` cubre secretos **antes** del primer commit (`.env*`, `*.pem`, `*.key`, data, logs)
- [ ] Escaneo de secretos en cada push **y sobre todo el historial** (betterleaks)
- [ ] Auditoría de CVEs de las dependencias, semanal aunque el repo no cambie (pip-audit)
- [ ] La auditoría mira **lo que realmente corre en el server**, no lo que CI resuelve
- [ ] Dependabot: alerts + security updates + malware alerts ON; version updates **solo para actions**
- [ ] Cada workflow declara `permissions:` mínimos (`contents: read`; sumar solo lo que use)
- [ ] Cero secrets de Actions si se puede (solo el `github.token` automático)
- [ ] Imágenes Docker de terceros pineadas **por digest SHA**, no por tag
- [ ] Los hallazgos llegan **al lugar donde el equipo mira** (issue en el tablero), no a la pestaña Actions
- [ ] Accesos con escritura: los mínimos, con 2FA confirmado, revisados cada tanto
- [ ] Sabés qué protección de `main` tenés (y cuál NO, por el plan)

---

## 1. `.gitignore` de secretos, primero

**Qué:** ignorar secretos y datos antes del primer commit.

💡 **Por qué:** lo que entra al historial de git no sale con un `rm`. Un secreto commiteado hay
que **redactarlo y reescribir el historial**, y rotarlo igual porque ya se considera quemado.

**Cómo (bloque base):**
```gitignore
# Secretos
.env
.env.*
!.env.example
*.pem
*.key
# Datos y salidas del server
data/*
logs/
*.log
tmp/
```
Convención: cada secreto tiene su `.example` versionado con el valor `cambiame`, así el
escáner sabe que es un placeholder y un clon limpio sabe qué variables necesita.

## 2. Escaneo de secretos: betterleaks (repo + historial)

**Qué:** correr betterleaks en cada push/PR, con `fetch-depth: 0` para escanear TODO el historial.
Usa BPE para bajar los falsos positivos respecto de gitleaks.

💡 **Por qué:** escanear solo el diff deja pasar lo que ya entró. El historial es la superficie.

**Cómo:**
```yaml
# .github/workflows/betterleaks.yml
name: betterleaks
on: [push, pull_request]
permissions:
  contents: read
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - name: betterleaks (repo + historial)
        run: |
          docker run --rm -v "$PWD:/repo" \
            -e GIT_CONFIG_COUNT=1 -e GIT_CONFIG_KEY_0=safe.directory -e GIT_CONFIG_VALUE_0=/repo \
            ghcr.io/betterleaks/betterleaks@sha256:<DIGEST> git /repo -v --config /repo/.betterleaks.toml
```
Falsos positivos → `.betterleaks.toml` con allowlist **por regex y con el porqué al lado**
(placeholders tipo `cambiame`, rutas `.example`). Nunca desactivar una regla entera.

## 3. Auditoría de CVEs: pip-audit, semanal

**Qué:** `pip-audit` en cada push/PR **y en un cron semanal**.

💡 **Por qué:** la base de CVEs cambia aunque el repo no. Un repo quieto un mes puede
tener tres CVEs nuevos sin que nadie haya tocado nada. El cron es lo que lo detecta.

**Modo observación (`continue-on-error: true`):** avisa, no bloquea PRs. Un CVE en una
transitiva no debería frenar a alguien que está tocando otra cosa. Ver §8 para que el aviso
llegue igual.

## 4. Auditar lo que corre en el server, no lo que CI resuelve

💡 **Por qué (la lección grande):** con `requirements.txt` con rangos (`openai>=3.3,<4`),
CI instala **lo último** dentro del rango. El server corre la imagen que se buildeó hace
meses. pip-audit daba verde con `openai 3.9` mientras el server corría `3.3.1`.
Peor: lo que el `Dockerfile` instala **fuera** de requirements (wheels de NVIDIA,
`onnxruntime-gpu` pineado a mano) no estaba en ningún archivo → **nunca se auditó**.

**Cómo:** una foto exacta del contenedor vivo, versionada en el repo:
```bash
docker exec <contenedor-api> pip freeze > eval_set/requirements-server.txt
```
- Se regenera **después de cada rebuild** de la imagen (paso del runbook de deploy).
- Chequeo de deriva en el runbook de salud (avisa si la foto quedó vieja):
  ```bash
  docker exec <contenedor-api> pip freeze | diff -q - eval_set/requirements-server.txt || echo "DERIVA: regenerar la foto"
  ```
- pip-audit la audita como segundo objetivo, y tolera que falte:
  ```yaml
      - name: pip-audit del server (solo avisa)
        id: server
        continue-on-error: true
        run: |
          if [ ! -f eval_set/requirements-server.txt ]; then
            echo "status=sin_foto" >> "$GITHUB_OUTPUT"; exit 0
          fi
          pip-audit --no-deps --desc --format markdown -r eval_set/requirements-server.txt | tee hallazgos.md
          echo "status=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"
  ```
- **Beneficio extra:** el `git diff` del archivo dice exactamente qué cambió en el server
  entre dos rebuilds.

⚠️ Footguns medidos:
- `--no-deps` **no evita** que pip-audit resuelva las dependencias (verificado). Está bien
  dejarlo, pero no confiar en que "solo lee la lista".
- No se puede correr localmente en Windows sobre un freeze de Linux: `uvloop` (que arrastra
  `uvicorn[standard]`) no compila en Windows. La prueba real es el log de CI.

## 5. Dependabot: qué prender y qué no

En repo **privado** el *Dependency graph* viene apagado; sin él nada funciona. Todo esto es
gratis en plan Free. `Settings → Advanced Security`:

| Opción | Prender | Qué hace |
|---|---|---|
| Dependency graph | ✅ | inventario de deps; base de todo lo demás |
| Dependabot alerts | ✅ | avisa CVEs, solo avisa |
| Dependabot malware alerts | ✅ | avisa si una dep fue tomada (supply chain) |
| Dependabot security updates | ✅ | abre PR **solo** cuando hay CVE con fix |
| Grouped security updates | ✅ | junta varios fixes en un PR |
| Dependabot version updates | ✅ pero **solo `github-actions`** | ver abajo |
| Dependabot on self-hosted runners | ❌ | correría en tu propia infra |
| Automatic dependency submission | ❌ | solo Maven/Gradle |

💡 **Por qué version updates NO para pip:** Dependabot respeta el techo pero **sube el piso**
(`openai>=3.3,<4` → `>=3.9.0,<4`). Si el server corre 3.3.1, mergear eso hace que CI instale
3.9 y **reabre la brecha CI≠server** que la foto del §4 cerró. Las versiones del server las
decide un rebuild deliberado, no un bot. Para actions sí paga: son pines exactos (`@v7`) y
nadie más los mira.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
    open-pull-requests-limit: 3
```

**Qué hace Dependabot sin permiso:** crea ramas y abre PRs (y cada PR dispara tus workflows).
**Qué NO hace nunca:** mergear. Y si el deploy es manual (`git pull` + restart), un PR
mergeado tampoco toca el server hasta que alguien rebuildea.

⚠️ Footguns medidos:
- **Cerrar un PR de Dependabot sin merge = "esta versión no la quiero"**: no la vuelve a
  proponer hasta que salga la siguiente major. Borrar la rama desde la UI cierra el PR igual.
- Si sacás un ecosistema del yml, **sus PRs viejos NO se cierran solos** (Dependabot ya no
  visita ese ecosistema). Cerrarlos a mano con motivo: `gh pr close N --delete-branch --comment "..."`.
- Los PRs no entran al tablero de Projects si el auto-add filtra `is:issue`. Es correcto:
  el tablero es *qué hacer*, un PR es *trabajo hecho esperando revisión*.

## 6. Permisos mínimos en cada workflow

```yaml
permissions:
  contents: read        # base para todos
  issues: write         # SOLO en el workflow que abre issues
```
💡 **Por qué:** el `github.token` de un run hereda lo que el workflow declare. Un workflow
sin `permissions:` explícito puede tener escritura en todo el repo. Si una action de terceros
se compromete, el daño es exactamente lo que le diste.

## 7. Cero secrets de Actions, imágenes por digest

- **Secrets:** el ataque clásico a CI es robar los secrets del repo. Si `gh secret list` está
  vacío y solo se usa `${{ github.token }}` (muere con cada run), **no hay botín**.
- **Actions oficiales de GitHub** (`actions/checkout`, `actions/setup-python`) por tag (`@v7`)
  es aceptable. **Actions de terceros: pinear por SHA** del commit, siempre.
- **Imágenes Docker de terceros: por digest** (`imagen@sha256:...`), nunca `:latest` ni `:v1`.
  Un tag lo pueden mover por debajo; un digest no.

## 8. Que el aviso llegue a donde el equipo mira

💡 **Por qué:** un radar que suena en una habitación vacía no existe. pip-audit corría cada
lunes y dejaba el resultado en la pestaña Actions; nadie entraba. El problema no era de
severidad (bloquear o no) sino de **entrega**.

**Cómo:** el run programado, si hay hallazgos, abre un issue con **título fijo** y etiqueta;
si ya existe uno abierto, comenta ahí (nunca uno por semana). El issue cae en el tablero por
la misma puerta que todo lo demás.
```yaml
      - name: avisar en el tablero si el server tiene CVEs
        if: github.event_name == 'schedule' && steps.server.outputs.status != '0' && steps.server.outputs.status != 'sin_foto'
        env:
          GH_TOKEN: ${{ github.token }}
          TITULO: "[pip-audit] dependencias del server con CVE"
        run: |
          cuerpo="$(printf 'Corrido el %s.\n\n' "$(date +%F)"; cat hallazgos.md)"
          num="$(gh issue list --state open --limit 100 | grep -F "$TITULO" | cut -f1 | head -1)"
          if [ -n "$num" ]; then gh issue comment "$num" --body "$cuerpo"
          else gh issue create --title "$TITULO" --label seguridad --body "$cuerpo"; fi
```
Requiere `issues: write` en ese workflow, y que el Project tenga **Auto-add** para `is:issue is:open`.

La alternativa "sacar `continue-on-error` del run programado" manda un mail a quien tocó el
cron por última vez: frágil y sin rastro en el tablero.

## 9. Accesos y protección de `main`

- **Colaboradores con escritura = puertas.** Los mínimos. Confirmar **2FA** en cada cuenta
  (GitHub no lo muestra; hay que preguntar). Podar los que ya no participan.
- **Branch protection / Rulesets en repo privado requieren GitHub Pro** (o repo público).
  En Free, cualquiera con write puede `push --force` a `main` y ningún check lo frena.
  Consecuencia honesta: en Free, tests y radares son **radar, no barrera**. `main` puede
  estar en rojo días sin que nada lo impida (pasó: 3 días por una función que superó el tope
  del radar). Compensación: disciplina de equipo + mirar el estado de `main` al arrancar.

## 10. Radares de código como CI (deuda cognitiva)

No es seguridad estricta pero corre en el mismo lugar y falla igual:
- `medir_comentarios.py`: comentarios máx 2 líneas (nº de issue + invariante; la historia va al issue).
- `medir_funciones.py`: funciones bajo un tope de líneas/anidamiento; si se pasa, **partirla en
  etapas nombradas**, no subir el tope. Excepciones explícitas en una lista, con el issue.

💡 **Por qué:** una regla en prosa se olvida; una regla que pone `main` en rojo, no.

---

## Lo que este nivel NO cubre

Todo lo anterior es el repo y el CI. Son otra capa y otra checklist:
- **API/server:** autenticación (API key + allowlist), TLS, rate limit, RLS/permisos por fila.
- **Infra:** qué puertos bindean a `127.0.0.1` vs LAN, qué corre como root, backups.
- **Datos:** qué información sensible entra a logs y a embeddings.
