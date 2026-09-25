# CONTEXTO PARA IA — AdminLinux · UTU · Clase 10

> 📋 Copiá TODO este archivo y pegalo en tu chatbot (Gemini/ChatGPT/Copilot) al inicio.

## Quién soy y qué estoy haciendo

Soy estudiante de "Administración de Sistemas Linux" (UTU, 2026), en la **Clase 10:
systemd + journalctl (servicios y logs)**. Trabajo en una **VM Ubuntu** (mis compañeros
usan Arch o Fedora: las tres valen). Ya vi pipes/filtros (C2), grep/regex (C3), sed (C4),
awk (C5), find/permisos (C6), red (C7) y procesos/señales/kill (C9).

**Reglas para vos, IA:**
- Explicame **cada directiva y cada flag** antes de que corra el comando.
- Si me proponés un `.service`, decime **qué hace cada línea** y hacémelo pasar por
  `systemd-analyze verify` **antes** de habilitarlo.
- **No me pongas `User=root`** si el servicio no lo necesita, ni secretos en
  `Environment=`.
- Quiero **aprender**, no copiar. Guiame.

## Lo que estoy viendo

**systemd** es el **PID 1**: el primer proceso que arranca el kernel. Arranca todo en el
orden correcto, lo vigila y lo reinicia si se cae. Es el "init" moderno (reemplaza SysV).

**Units** = la ficha de cada cosa que systemd maneja. Tipos: `.service` (un programa),
`.target` (un grupo/modo, reemplaza los runlevels), `.timer` (alarma, tipo cron),
`.socket` (un puerto), `.mount` (un montaje).

**Dónde viven**: `/usr/lib/systemd/system/` son las del **paquete** (NO se tocan) y
`/etc/systemd/system/` son las **mías** (ganan si el nombre se repite). Para retocar una
del paquete sin pisarla: `systemctl edit <unit>` (crea un override).

**systemctl**:

| Comando | Qué hace |
|---|---|
| `status` / `is-active` / `is-enabled` | ver estado (no necesita sudo) |
| `start` / `stop` / `restart` | ahora mismo |
| `reload` | releer config sin cortar (necesita `ExecReload=`) |
| `enable` / `disable` | al arranque (crea/borra un symlink) |
| `enable --now` | enable + start de una |
| `mask` / `unmask` | bloquear del todo (symlink a `/dev/null`) |
| `daemon-reload` | releer los archivos de unit desde el disco |
| `list-units` / `list-unit-files --state=enabled` / `--failed` | listar |

**enable ≠ start**: `enable` = "al boot", `start` = "ahora". Son independientes.

**daemon-reload**: si edito un `.service` y no lo corro, systemd sigue con la versión
vieja en memoria.

**Leer un `systemctl status`**: `Loaded` (¿encontró el archivo? ¿de dónde? ¿enabled?),
`Active` (estado + desde cuándo), `Main PID`, `Tasks`, `Memory`, `CGroup` (el árbol de
procesos) y las **últimas líneas del log** (ahí suele estar la causa).

**Anatomía de un `.service`**: `[Unit]` (Description, After, Requires, Wants),
`[Service]` (Type, ExecStart con ruta ABSOLUTA, ExecReload, Restart, RestartSec, User,
Group, WorkingDirectory, Environment, EnvironmentFile), `[Install]` (WantedBy).

**Targets**: `multi-user.target` (servidor, sin gráfico) vs `graphical.target`.
`systemctl get-default` / `set-default` / `isolate rescue.target`.

**Timers**: `respaldo.timer` (CUÁNDO) dispara `respaldo.service` (QUÉ). `OnCalendar=`,
`Persistent=true`, `systemctl list-timers`. Ventajas sobre cron: logging integrado,
dependencias, hardening. (cron a fondo lo vemos en la Clase 12.)

**journalctl**: el log centralizado y estructurado (`/var/log/journal/` si es
persistente, `/run/log/journal/` si es volátil).

| Flag | Para qué |
|---|---|
| `-u <unit>` | solo ese servicio |
| `-n 50` | últimas 50 líneas |
| `-f` | seguir en vivo |
| `-p err` | prioridad err o peor (0 emerg … 7 debug) |
| `-b` / `-b -1` | boot actual / anterior |
| `-k` | solo kernel |
| `--since` / `--until` | ventana de tiempo |
| `-o json-pretty` / `-o short-iso` | formato de salida |
| `--disk-usage` / `--vacuum-time=7d` | cuánto ocupa / podar |
| `--no-pager` | sin `less` (obligatorio en scripts) |

## Ejemplos que estoy usando

```bash
systemctl status nginx
sudo systemctl enable --now nginx
sudo systemctl daemon-reload && sudo systemctl restart tienda
journalctl -u tienda -n 50 --no-pager
journalctl -b -p err --no-pager
journalctl --since "09:00" --until "09:30" -u nginx
systemd-analyze verify /etc/systemd/system/tienda.service
systemd-analyze blame | head
```

## Flujo de diagnóstico que me enseñaron ("el servicio no levanta")

1. `systemctl status <unit>` → mirar `Loaded` / `Active` / código de salida.
2. `journalctl -u <unit> -n 50 --no-pager` → **acá está la causa real**.
3. `systemd-analyze verify <archivo>` → ¿la unit está bien escrita?
4. Corregir en `/etc/systemd/system/`.
5. `sudo systemctl daemon-reload` (sin esto el cambio no existe).
6. `sudo systemctl restart <unit>` y volver al paso 1.

Códigos `2xx` (`203/EXEC`, `200/CHDIR`, `217/USER`, `226/NAMESPACE`) son **de systemd**:
el programa ni arrancó. Otros códigos (`1`, `137`) son de la app.

## Datos con los que trabajo

- `datos/journal_ejemplo.log`: volcado chico de journal (host `srv-demo`).
- `datos/tienda-api.service`: una unit con **errores a propósito** (para criticarla).
- Mi propia VM: `systemctl`, `journalctl` (leer no necesita sudo).

## El reto de la bandera (Ansible + systemd)

En `playbooks/` hay un reto práctico. Con `ansible-playbook armar_reto.yml -l local`
despliego mi estación de guardia (instala `gawk`, deja `datos/reto/journal_reto.log` y
`datos/reto/unidades_reto.txt`, y una `MISION.txt`). Después hago el postmortem de
`web01`: R1 = cuántas líneas dicen exactamente `Failed with result`, R2 = qué unidad
juntó más de esas (sin el `.service`), R3 = cuántos `Scheduled restart job` programó
systemd de esa unidad. Armo `BANDERA{<R1>-<R2>-<R3>}` y la verifico con
`ansible-playbook comprobar_bandera.yml -e "bandera=..."`.

Si te pido ayuda con esto: **explicame el comando** y **hacéme verificar** con
`head`/`less` sobre una muestra, **no me des la bandera hecha**. El playbook es
**multi-distro** (usa `apt`, `pacman` o `dnf` según mi distro) — eso es **correcto**, no
un error.

## Concepto de seguridad de hoy

- `User=`/`Group=`: un servicio que no necesita root **no corre como root**. Si no hay
  usuario, `DynamicUser=yes`.
- `NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`,
  `ReadWritePaths=<solo lo que necesita escribir>`.
- **Antipatrón**: secretos en `Environment=` (los lee cualquiera con `systemctl show` o
  `/proc/<pid>/environ`) → van en `EnvironmentFile=` con permisos `0600`.
- `systemd-analyze security <unit>` te pone nota de 0 a 10 (más bajo = más seguro).

## Trampa multi-distro (nombres de unit)

| Servicio | Debian/Ubuntu | Arch | Fedora/RHEL |
|---|---|---|---|
| SSH | `ssh.service` | `sshd.service` | `sshd.service` |
| Apache | `apache2.service` | `httpd.service` | `httpd.service` |
| cron | `cron.service` | `cronie.service` | `crond.service` |

Si "no existe" una unit: `systemctl list-units '*ssh*'`.

## Preguntas típicas

- "Explicame este `.service` línea por línea: `...`"
- "¿Por qué mi cambio en el .service no tiene efecto?" (respuesta: falta `daemon-reload`)
- "¿Cuál es la diferencia entre `enable` y `start`?"
- "Escribime un `.service` para X, pero explicámelo y decime cómo verificarlo antes de
  habilitarlo."
- "Mi servicio da `203/EXEC`, ¿qué miro?"

Recordá: explicame cada directiva, nada de `User=root` porque sí, y haceme verificar
antes de habilitar.
