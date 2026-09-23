# Seguridad del repo y del CI

**Alcance:** checklist para arrancar o auditar un repo. Cada punto trae *qué*, 💡 *por qué* y
*cómo* (copiable).

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
- [ ] El server clona con una **deploy key de solo lectura**, no con un token personal
- [ ] Todo secreto que se pegó en un chat (con personas o con IA) se rotó

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
Un secreto pegado en un chat, un issue o una captura cuenta igual que uno commiteado:
**quemado, se rota** (ver `Trabajar-con-IA.md` §7).

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
tener CVEs nuevos sin que nadie haya tocado nada. El cron es lo que lo detecta.

**Modo observación (`continue-on-error: true`):** avisa, no bloquea PRs. Un CVE en una
transitiva no debería frenar a alguien que está tocando otra cosa. Ver §8 para que el aviso
llegue igual.

## 4. Auditar lo que corre en el server, no lo que CI resuelve

💡 **Por qué (el más importante):** con `requirements.txt` con rangos (`libx>=1.3,<2`),
CI instala **lo último** dentro del rango, pero el server corre la imagen que se buildeó hace
meses. pip-audit puede dar verde con `libx 1.9` mientras el server corre `1.3.1`.
Peor: lo que el `Dockerfile` instala **fuera** de requirements (wheels pineados a mano) no
está en ningún archivo → **nunca se audita**.

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
- `--no-deps` **no evita** que pip-audit resuelva las dependencias. Está bien
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
(`libx>=1.3,<2` → `>=1.9.0,<2`). Si el server corre 1.3.1, mergear eso hace que CI instale
1.9 y **reabre la brecha CI≠server** que la foto del §4 cerró. Las versiones del server las
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

💡 **Por qué:** un radar que suena en una habitación vacía no existe. Un pip-audit semanal que
deja el resultado en la pestaña Actions no lo lee nadie. El problema no era de
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

## 9. Accesos y protección de `main`

- **Colaboradores con escritura = puertas.** Los mínimos. Confirmar **2FA** en cada cuenta
  (GitHub no lo muestra; hay que preguntar). Podar los que ya no participan.
- **Branch protection / Rulesets en repo privado requieren GitHub Pro** (o repo público).
  En Free, cualquiera con write puede `push --force` a `main` y ningún check lo frena.
  Consecuencia honesta: en Free, tests y radares son **radar, no barrera**. `main` puede
  estar en rojo días sin que nada lo impida. Compensación: disciplina de equipo + mirar el
  estado de `main` al arrancar.
- **El server no usa tu token.** Para que el server haga `git pull`, una **deploy key** SSH
  del repo, de solo lectura, con un alias en `~/.ssh/config`. Nunca un token personal en
  `~/.git-credentials` ni en la URL del remote.
  💡 Un token personal con scope `repo` en texto plano da escritura a **todos** los repos de
  esa persona a cualquiera que lea ese archivo; una deploy key da lectura a uno solo.
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/<proyecto>_deploy -N "" -C "<proyecto>-server"
  # la .pub va a Settings → Deploy keys (sin "Allow write access")
  # ~/.ssh/config:
  #   Host github.com-<proyecto>
  #     HostName github.com
  #     IdentityFile ~/.ssh/<proyecto>_deploy
  #     IdentitiesOnly yes
  git remote set-url origin git@github.com-<proyecto>:<owner>/<repo>.git
  ssh -T git@github.com-<proyecto>
  ```

---

## Lo que este nivel NO cubre

Todo lo anterior es el repo y el CI. Lo ya aprendido del server (compartido, salud, candados
de servicios internos) está en [`Operacion-server.md`](Operacion-server.md). Son otra capa:
- **API/server:** autenticación (API key + allowlist), TLS, rate limit, RLS/permisos por fila.
- **Infra:** qué puertos bindean a `127.0.0.1` vs LAN, qué corre como root, backups.
- **Datos:** qué información sensible entra a logs y a embeddings.
