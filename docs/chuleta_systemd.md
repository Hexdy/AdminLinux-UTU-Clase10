# Chuleta — systemd + journalctl

## Los 5 tipos de unit que importan
| Tipo | Es… | Ejemplo |
|---|---|---|
| `.service` | un programa que corre | `nginx.service` |
| `.target` | un grupo de units (un "modo") | `multi-user.target` |
| `.timer` | una alarma que dispara un `.service` | `respaldo.timer` |
| `.socket` | un puerto que escucha y levanta el servicio | `cockpit.socket` |
| `.mount` | un punto de montaje | `var-log.mount` |

## Dónde viven
| Ruta | Quién la escribe | ¿La tocás? |
|---|---|---|
| `/usr/lib/systemd/system/` | el paquete (`apt`/`pacman`/`dnf`) | ❌ nunca |
| `/etc/systemd/system/` | **vos** (gana si el nombre se repite) | ✅ sí |
| `/run/systemd/system/` | temporales (se van al reiniciar) | ❌ |

## systemctl
```bash
systemctl status nginx            # estado + últimas líneas de log (sin sudo)
systemctl is-active nginx         # active / inactive / failed  (para scripts)
systemctl is-enabled nginx        # enabled / disabled / masked / static

sudo systemctl start nginx        # AHORA
sudo systemctl stop nginx
sudo systemctl restart nginx      # stop + start (PID nuevo, corta conexiones)
sudo systemctl reload nginx       # releer config sin cortar (necesita ExecReload=)
sudo systemctl reload-or-restart nginx

sudo systemctl enable nginx       # AL ARRANQUE (crea un symlink)
sudo systemctl disable nginx
sudo systemctl enable --now nginx # enable + start de una

sudo systemctl mask bluetooth     # bloquear del todo (symlink a /dev/null)
sudo systemctl unmask bluetooth

systemctl list-units --type=service --all
systemctl list-unit-files --state=enabled
systemctl --failed                # ¿qué está roto en esta máquina?
sudo systemctl daemon-reload      # después de EDITAR un archivo de unit
sudo systemctl edit nginx         # override en /etc/ (no pisa el del paquete)
```

## enable vs start
| | NO arranca al boot | SÍ arranca al boot |
|---|---|---|
| **NO corre ahora** | `disabled` + `stopped` | `enabled` + `stopped` |
| **SÍ corre ahora** | `disabled` + `running` ⚠️ | `enabled` + `running` ✅ |

`enable` = al arranque · `start` = ahora. Son **independientes**.

## Leer un `systemctl status`
| Línea | Qué te dice |
|---|---|
| `Loaded:` | si encontró el archivo, **de dónde** y si está `enabled` |
| `Active:` | `running` / `exited` / `waiting` / `dead` / `failed` + desde cuándo |
| `Main PID:` | el proceso principal (el de la Clase 9) |
| `Tasks:` / `Memory:` | cuántos hilos y cuánta RAM consume |
| `CGroup:` | el árbol de procesos del servicio (acá ves los workers) |
| últimas líneas | 🔎 **la causa suele estar acá** |

Códigos `2xx` = de **systemd** (el programa ni arrancó): `203/EXEC` (ruta mala o sin
permiso de ejecución), `200/CHDIR` (`WorkingDirectory=` no existe), `217/USER` (el
`User=` no existe), `226/NAMESPACE` (chocó con el hardening). Otros (`1`, `137`) son
de la app.

## Anatomía de un `.service`
```ini
[Unit]
Description=Qué es esto
After=network-online.target      # ORDEN (no implica que ande)
Wants=network-online.target      # dependencia blanda (si falla, sigo)
# Requires=postgresql.service    # dependencia dura (si falla, me caigo)

[Service]
Type=simple                      # simple | exec | forking(+PIDFile=) | oneshot | notify
ExecStart=/usr/local/bin/app     # RUTA ABSOLUTA, siempre
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure               # no | on-failure | always
RestartSec=5
User=miapp                       # <- SEGURIDAD. O DynamicUser=yes
WorkingDirectory=/opt/miapp
EnvironmentFile=/etc/miapp.env   # secretos acá (0600), NO en Environment=

[Install]
WantedBy=multi-user.target       # solo la lee  systemctl enable
```

## Hardening (security by design)
```ini
NoNewPrivileges=true     # no puede escalar ni con binarios SUID
ProtectSystem=strict     # todo el disco en solo-lectura...
ReadWritePaths=/var/lib/miapp   # ...salvo lo que necesita escribir
ProtectHome=true         # no ve /home, /root ni /run/user
PrivateTmp=true          # /tmp propio y aislado
```
`systemd-analyze security <unit>` → nota de 0 a 10 (**más bajo = más seguro**).

## Targets
```bash
systemctl get-default                          # graphical.target / multi-user.target
sudo systemctl set-default multi-user.target   # servidor: sin entorno gráfico
sudo systemctl isolate rescue.target           # ir a rescate AHORA (corta sesiones)
systemctl list-dependencies multi-user.target
```

## Timers
```bash
systemctl list-timers --all
systemd-analyze calendar "*-*-* 03:30:00"      # ¿cuándo dispararía?
sudo systemctl enable --now respaldo.timer     # se habilita el .timer, no el .service
```
`OnCalendar=` (cuándo) · `Persistent=true` (si estaba apagada, corre al prender) ·
`RandomizedDelaySec=` (que no peguen todos juntos).

## journalctl
| Comando | Para qué |
|---|---|
| `journalctl -u nginx` | solo esa unit |
| `journalctl -u nginx -n 50 --no-pager` | últimas 50, sin pager |
| `journalctl -u nginx -f` | en vivo (Ctrl+C sale) |
| `journalctl -b` / `-b -1` | boot actual / anterior |
| `journalctl -p err` | prioridad `err` o peor |
| `journalctl --since "09:00" --until "09:30"` | ventana de tiempo |
| `journalctl --since yesterday` | desde ayer |
| `journalctl -k` | solo kernel (el viejo `dmesg`) |
| `journalctl -o json-pretty` | salida estructurada (para scripts) |
| `journalctl -o short-iso` | fecha ISO (ordenable, comparable) |
| `journalctl --disk-usage` | cuánto ocupa el journal |
| `sudo journalctl --vacuum-time=7d` | podar lo más viejo de 7 días |

**Prioridades**: `0 emerg` · `1 alert` · `2 crit` · `3 err` · `4 warning` ·
`5 notice` · `6 info` · `7 debug`.

**Persistente vs volátil**: si existe `/var/log/journal/` el journal **sobrevive al
reinicio**; si no, vive en `/run/` y se borra. Para hacerlo persistente:
`Storage=persistent` en `/etc/systemd/journald.conf` +
`sudo systemctl restart systemd-journald`. Sin eso, el log del crash de anoche **no
está**.

## Diagnóstico ("el servicio no levanta")
```bash
systemctl status <unit>
journalctl -u <unit> -n 50 --no-pager     # <- la causa real casi siempre acá
systemd-analyze verify /etc/systemd/system/<unit>
# corregís el archivo...
sudo systemctl daemon-reload              # SIN ESTO TU CAMBIO NO EXISTE
sudo systemctl restart <unit>
```
Arranque lento: `systemd-analyze` · `systemd-analyze blame` ·
`systemd-analyze critical-chain`.

## 🐧 Trampa multi-distro: el mismo servicio, distinto nombre
| Servicio | Debian / Ubuntu | Arch | Fedora / RHEL |
|---|---|---|---|
| SSH | `ssh.service` | `sshd.service` | `sshd.service` |
| Apache | `apache2.service` | `httpd.service` | `httpd.service` |
| cron | `cron.service` | `cronie.service` | `crond.service` |
| Firewall | `ufw.service` | `nftables.service` | `firewalld.service` |

¿"No existe"? → `systemctl list-units '*ssh*'`

## Errores típicos
- Editar el `.service` y no correr `daemon-reload` → systemd sigue con la copia vieja.
- `ExecStart=npm start` (ruta relativa) → `203/EXEC`. Va la absoluta.
- `enable` sin `start` → "lo habilité y no anda" (arranca recién al reiniciar).
- `Type=forking` sin `PIDFile=` → systemd no sabe a quién vigilar.
- Usar `apt` con `sshd.service` de Fedora (o al revés): **mirá el nombre de tu distro**.
- `grep` sensible a mayúsculas: `Failed with result` ≠ `failed with result`.
