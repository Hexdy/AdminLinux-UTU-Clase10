# Playbooks de Ansible — AdminLinux Clase 10 (systemd + el reto de la bandera)

Un **escenario práctico de SRE**: desplegás un playbook de Ansible que te arma la
estación de guardia (instala `gawk`, deja el material del incidente, prepara la
misión) y después, con lo que viste de **systemd/journalctl**, hacés el
**postmortem de `web01`**. Con las 3 respuestas armás una **bandera** y la
verificás con otro playbook. Aprendés Ansible **haciendo**, y análisis de logs
resolviendo un problema real (no ejercicios de juguete).

> 🐧 Con una sola VM alcanza: todo corre contra `localhost` (grupo `[local]`).
> No es hacking: es la guardia normal de un SRE (leer el journal y entender por
> qué se cayó un servicio).

## Paso 0 — ¿Tenés todo? (te dice qué falta y cómo instalarlo)

```bash
cd playbooks
bash comprobar_requisitos.sh
```

Detecta tu distro y, si falta algo (Ansible, Python, gawk), te da el comando exacto
para **apt**, **pacman** o **dnf**. De paso te dice cómo se llama el servicio de
**SSH en tu distro** (`ssh.service` vs `sshd.service`), que es la trampa clásica
de esta clase.

## Paso 1 — Desplegá el reto (armá la estación de guardia)

```bash
# Simular primero (no toca nada, buena costumbre):
ansible-playbook armar_reto.yml -l local --check --diff

# Aplicar de verdad (deja datos/reto/ listo):
ansible-playbook armar_reto.yml -l local
#   ↳ si al instalar gawk te pide contraseña de sudo, agregá  -K
```

Atajo que hace el preflight + el armado:

```bash
bash correr.sh
```

Esto deja tu misión en `../datos/reto/MISION.txt`. Leela:

```bash
cat ../datos/reto/MISION.txt
```

> 🔒 El playbook **no toca ningún servicio de tu máquina**. Lo único que hace con
> `systemctl` es un `is-enabled` de lectura, para mostrarte el nombre de la unit
> de SSH en tu distro. No arranca, no habilita, no para nada.

## Paso 2 — Hacé el postmortem de web01

Te dejaron en `datos/reto/`:

- `journal_reto.log` — volcado de `journalctl -b -o short-iso --no-pager`.
- `unidades_reto.txt` — volcado de `systemctl list-units` / `list-unit-files` / `list-timers`.
- `nota_equipo.txt` — la nota del compañero (quedó en `0600`).

El jefe de guardia quiere 3 números:

1. **R1** — cuántas líneas contienen **exactamente** `Failed with result`.
2. **R2** — qué **unidad** acumuló más de esas líneas (**sin** el `.service`).
3. **R3** — cuántos `Scheduled restart job` programó systemd **de esa unidad**.

Armá la bandera con el formato exacto: `BANDERA{<R1>-<R2>-<R3>}`.

> 💡 Los comandos exactos NO están acá a propósito: la idea es que los armes vos
> (el apunte de la clase te da todas las piezas). Dos trampas: hay líneas con
> `failed with result 'dependency'` en **minúscula** (caídas por dependencia, **no**
> cuentan) y líneas `Failed to start ...` (el resumen, tampoco cuenta).
> Si le pedís ayuda a una IA, **entendé el comando** antes de correrlo.

## Paso 3 — Verificá tu bandera con Ansible

```bash
ansible-playbook comprobar_bandera.yml -e "bandera=BANDERA{...}"
```

- ✅ correcta → 🏁 trofeo.
- ❌ incorrecta → volvé a mirar cada número (y releé las dos trampas).

El repo **no guarda la bandera en texto plano**: solo su huella **SHA-256** en
`secretos/bandera.sha256`. Es la misma idea con la que se guardan las contraseñas
(nunca el secreto, solo su hash). **Security by design.**

## Probá la idempotencia (lo más importante de Ansible)

Corré `armar_reto.yml` **dos veces seguidas**: la 1ª hace cambios (`changed`), la 2ª
deja todo en `ok` (`changed=0`, ya estaba como debía). Ese es el corazón de IaC:
describís el **estado deseado**, no pasos sueltos. Por eso los `command` del
playbook llevan `changed_when: false`: leer no es cambiar.

## 🔀 Multi-distro: el mismo playbook en 3 familias

| | Debian / Ubuntu | Arch | Fedora / RHEL |
|---|---|---|---|
| Gestor | `apt` | `pacman` | `dnf` |
| Paquete awk | `gawk` | `gawk` | `gawk` |
| Detección | `ansible_os_family == 'Debian'` | `'Archlinux'` | `'RedHat'` |
| **Unit de SSH** | **`ssh.service`** | **`sshd.service`** | **`sshd.service`** |
| Unit de Apache | `apache2.service` | `httpd.service` | `httpd.service` |
| Unit de cron | `cron.service` | `cronie.service` | `crond.service` |

El playbook detecta la familia con `ansible_os_family` y el módulo genérico
`package` elige el gestor por vos. La instalación va envuelta en **`block`/`rescue`**:
si no hay sudo o no hay red, te avisa y **sigue** en vez de dejarte el laboratorio
a medio armar.

## Qué hay acá

| Archivo | Qué es |
|---|---|
| `armar_reto.yml` | Despliega la estación: gawk + material del incidente + misión (tri-distro, idempotente). |
| `comprobar_bandera.yml` | Verifica tu bandera por hash (sin exponer el secreto). |
| `comprobar_requisitos.sh` | Preflight: detecta tu distro y te dice qué instalar. |
| `correr.sh` | Atajo: preflight + arma el reto. |
| `templates/mision.txt.j2` | Plantilla Jinja de la misión (se personaliza con tus facts). |
| `reto/journal_reto.log` | El volcado del journal de la noche del incidente. |
| `reto/unidades_reto.txt` | Estado de las units de `web01` a las 08:05. |
| `reto/nota_equipo.txt` | Nota del equipo (se deja en 0600). |
| `secretos/bandera.sha256` | Solo la **huella** de la bandera (no el secreto). |
| `inventory.ini` / `ansible.cfg` | Qué máquinas administra Ansible + config local. |

## ⚠️ Seguridad

- Ningún playbook usa `chmod 777`, contraseñas en texto plano ni secretos versionados.
- La nota con valor queda en **0600** (menor privilegio) por diseño.
- La bandera se guarda como **hash**, no en claro.
- Nada se arranca ni se habilita en tu máquina: el único `systemctl` es de **lectura**.
- Simulá (`--check --diff`) antes de aplicar algo que no conozcas.
