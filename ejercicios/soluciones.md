# Soluciones — Clase 10 (¡intentá antes de mirar!)

> ⚠️ Probá cada comando vos primero. Los de lectura (`status`, `is-active`,
> `journalctl`) **no necesitan `sudo`**: podés equivocarte sin romper nada.
>
> 🐧 Los ejemplos usan los nombres de **Debian/Ubuntu**. En Arch/Fedora cambiá
> `ssh` por `sshd`.

## 1) ¿SSH corre? ¿y arranca al boot?

```bash
systemctl is-active ssh      # -> active | inactive | failed
systemctl is-enabled ssh     # -> enabled | disabled | masked | static
```

Son **dos ejes distintos**: `is-active` habla del **ahora**, `is-enabled` del
**arranque**. Un servicio puede estar `active` y `disabled` a la vez (anda, pero si
reiniciás la máquina no vuelve).

## 2) Units en estado failed

```bash
systemctl --failed
# equivalente explícito:
systemctl list-units --type=service --state=failed
```

Si no hay nada roto, devuelve `0 loaded units listed.` — eso **también es una
respuesta**, y es la que querés ver en un servidor.

## 3) Cuántas units están habilitadas

```bash
systemctl list-unit-files --state=enabled --no-pager | wc -l
```

Ojo: `wc -l` cuenta también el encabezado y las líneas del resumen. Para contar solo
las units:

```bash
systemctl list-unit-files --state=enabled --no-legend --no-pager | wc -l
```

## 4) Target por defecto

```bash
systemctl get-default
```

En una VM de escritorio devuelve `graphical.target`. En un **servidor** va
`multi-user.target`:

```bash
sudo systemctl set-default multi-user.target
```

Por qué: el entorno gráfico se come RAM y CPU que no necesitás, y **más software
corriendo = más superficie de ataque**.

## 5) Arranque lento

```bash
systemd-analyze                      # total: kernel + userspace
systemd-analyze blame | head -n 3    # los 3 servicios más lentos
systemd-analyze critical-chain       # la cadena que REALMENTE demoró el arranque
```

`blame` te da los más lentos; `critical-chain` te dice cuáles de esos estaban en el
camino crítico (un servicio lento que arranca en paralelo no demora nada).

## 6) Últimas 20 líneas del log de SSH

```bash
journalctl -u ssh -n 20 --no-pager
```

`--no-pager` evita que te abra `less`. En scripts es **obligatorio** (si no, el script
se queda esperando).

## 7) Errores del boot actual y del anterior

```bash
journalctl -b -p err --no-pager
journalctl -b -1 -p err --no-pager
```

Si el segundo te dice `Specified boot ID ... not found`, tu journal es **volátil**: vive
en `/run/log/journal/` y se borra al reiniciar. Se arregla creando
`/var/log/journal/` o poniendo `Storage=persistent` en `/etc/systemd/journald.conf` y
reiniciando `systemd-journald`. **Sin eso, el log del crash de anoche no existe.**

```bash
ls -d /var/log/journal 2>/dev/null && echo "persistente" || echo "volatil"
```

## 8) Contar fallos en `journal_ejemplo.log`

```bash
grep -c "Failed with result" datos/journal_ejemplo.log
# -> 3
```

Y la unidad que los generó:

```bash
grep "Failed with result" datos/journal_ejemplo.log \
  | grep -o '[a-z0-9-]*\.service' | sort | uniq -c | sort -rn
#    3 demo-cola.service
```

Las 3 son de `demo-cola.service`: no encuentra su archivo de configuración, systemd la
reinicia 2 veces (`Scheduled restart job`) y a la 3ª corta con
`Start request repeated too quickly`.

> Ojo con `grep -i`: te sumaría líneas en minúscula que **no** son fallos propios.

## 9) Los 5 problemas de `tienda-api.service`

| # | Línea | Problema | Arreglo |
|---|---|---|---|
| 1 | `User=root` | 🔴 **Seguridad**: una RCE en la app = root de la máquina. Una API web no necesita root. | `User=tienda` (o `DynamicUser=yes`) |
| 2 | `Environment="DB_PASSWORD=..."` | 🔴 **Seguridad**: lo lee cualquiera con `systemctl show tienda-api` o mirando `/proc/<pid>/environ`. Además queda versionado. | `EnvironmentFile=/etc/tienda-api.env` con permisos `0600` |
| 3 | `Type=forking` sin `PIDFile=` | El servicio no arranca bien: systemd no sabe **a qué proceso vigilar** y se queda esperando. | `Type=simple` (Node no hace fork) |
| 4 | `ExecStart=node server.js` | Ruta **relativa** → `status=203/EXEC`. systemd no usa tu `PATH`. | `ExecStart=/usr/bin/node /opt/tienda/server.js` |
| 5 | `Restart=always` + `RestartSec=1` | Si la app falla al arrancar, entra en **loop** martillando el sistema (y el journal). | `Restart=on-failure`, `RestartSec=5` y `StartLimitIntervalSec=`/`StartLimitBurst=` en `[Unit]` |

Bonus: `After=network.target` no garantiza que haya **IP**; para algo que sale a la red
va `After=network-online.target` + `Wants=network-online.target`.

Versión corregida:

```ini
[Unit]
Description=API de la tienda
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
Type=simple
User=tienda
Group=tienda
WorkingDirectory=/opt/tienda
ExecStart=/usr/bin/node /opt/tienda/server.js
EnvironmentFile=/etc/tienda-api.env
Environment="TIENDA_PUERTO=8080"
Restart=on-failure
RestartSec=5

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/tienda

[Install]
WantedBy=multi-user.target
```

Verificación antes de habilitarla:

```bash
systemd-analyze verify /etc/systemd/system/tienda-api.service
sudo systemctl daemon-reload
sudo systemctl start tienda-api      # probás que ANDA...
systemctl status tienda-api
sudo systemctl enable tienda-api     # ...y recién ahí la habilitás
```

## 10) "Le puse `User=root` porque si no, no me anda"

Preguntas antes de aceptarlo:

- ¿**Qué** exactamente le falla sin root? Casi siempre es una de dos cosas: no puede
  **escribir** en un directorio (se arregla con `StateDirectory=` o
  `ReadWritePaths=`) o quiere **escuchar en un puerto < 1024** (se arregla con
  `AmbientCapabilities=CAP_NET_BIND_SERVICE` o poniéndolo detrás de un proxy).
- ¿Probaste con un usuario propio (`useradd --system --no-create-home tienda`) o con
  `DynamicUser=yes`?

Dos directivas de hardening para esa unit:

- `NoNewPrivileges=true` — el proceso **no puede escalar privilegios** ni aunque
  ejecute un binario SUID. Corta la escalada más común post-RCE.
- `ProtectSystem=strict` — monta **todo el disco en solo-lectura** para ese servicio,
  salvo lo que listes en `ReadWritePaths=`. Aunque lo comprometan, no te toca
  `/etc` ni `/usr`.

Y de yapa: `PrivateTmp=true` (un `/tmp` propio, para que no espíe ni le planten
archivos) y `ProtectHome=true` (no ve `/home` ni `/root`).

Para medirlo: `systemd-analyze security tienda-api.service` → nota de 0 a 10, **más
bajo = más seguro**.
