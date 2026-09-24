# AdminLinux · UTU · Clase 10 — systemd + journalctl (servicios y logs)

> Repo de la **Clase 10** de *Administración de Sistemas Linux* (UTU, 2026).
> Docente: **Ulyses García**. Repositorio **privado** de práctica.

Doble uso: 🤖 pegá `CONTEXTO_IA.md` en tu chatbot · 🧑‍💻 cloná y practicá `systemctl` /
`journalctl` sobre tu VM y sobre los `datos/`.

## 🚀 Arranque rápido

> 🔑 Cloná **fuera** de Google Drive / OneDrive. El sync corrompe `.git`.

```bash
git clone <url-de-este-repo> C:\src\AdminLinux-UTU-Clase10
cd C:\src\AdminLinux-UTU-Clase10

# Tu primer diagnóstico (en tu VM, sin sudo: status y journalctl leen nomás)
systemctl status ssh          # en Arch/Fedora es sshd
journalctl -u ssh -n 20 --no-pager
systemctl --failed            # ¿hay algo roto en esta máquina?
```

## 🗂️ Qué hay acá

| Carpeta / archivo | Qué es |
|---|---|
| `CONTEXTO_IA.md` | **Pegá esto en tu chatbot** para que te asista con contexto. |
| `datos/journal_ejemplo.log` | Volcado chico de un journal, para practicar sin tocar tu sistema. |
| `datos/tienda-api.service` | Una unit con **errores a propósito** (ejercicio 8). |
| `ejercicios/` | 9 retos de systemd/journalctl + soluciones (con salidas esperadas). |
| `docs/` | Chuleta de systemd + el reto de la bandera. |
| `playbooks/` | 🏁 **El reto de la bandera**: desplegás con Ansible (multi-distro) tu estación de guardia y hacés el postmortem de `web01`. |

## 🏁 El reto de la bandera (Ansible + systemd)

El plato fuerte de la clase vive en [`playbooks/`](playbooks/README.md). Es un
**escenario real de SRE**, no un ejercicio de juguete:

1. **Desplegás** tu estación con Ansible: `ansible-playbook armar_reto.yml -l local`
   (instala `gawk`, deja el material del incidente y te genera una `MISION.txt` personalizada).
2. **Hacés el postmortem** de `web01` sobre `datos/reto/`: cuántos fallos reales hubo,
   qué unidad falló más y cuántos reinicios le programó systemd a esa unidad.
3. **Armás la bandera** `BANDERA{<R1>-<R2>-<R3>}` y la **verificás** con
   `ansible-playbook comprobar_bandera.yml -e "bandera=BANDERA{...}"`.

Aprendés **Ansible haciendo** (idempotencia, módulos, facts, `block`/`rescue`) y
**lectura de logs resolviendo un problema real**. El **mismo** playbook corre en
**Ubuntu, Arch y Fedora** (detecta la distro solo, sin cambiar una línea).

> 🔒 La bandera no se guarda en texto plano: el repo solo tiene su **hash SHA-256**
> (igual que se guardan las contraseñas). *Security by design.*

> 🗣️ **La bandera NO se entrega por Pull Request**: cuando la verificás (✅), **se la
> decís al profe** en clase o por el canal del curso.

## 🧵 Hilo del curso

En la **Clase 9** viste los procesos: `ps`, `top`, señales, `kill`. Hoy subimos un
escalón: **quién arranca y vigila** esos procesos. `systemd` es el PID 1, el padre de
todo, y `journalctl` es su cuaderno de bitácora. Con esto pasás de *mirar* el sistema a
*administrarlo*.

## 🎯 Los 5 pilares

Hoy pega fuerte **Security by Design** (`User=`, `NoNewPrivileges=`, `ProtectSystem=`,
secretos fuera del unit) e **IA aplicada** (pedirle un `.service` a un LLM y
**verificarlo** con `systemd-analyze verify` antes de habilitarlo). Y el reto de
[`playbooks/`](playbooks/README.md) suma **IaC + Ansible**: desplegás el entorno **como
código**, idempotente y multi-distro.

## ⚠️ Seguridad

- Un servicio con `User=root` que no lo necesita convierte cualquier RCE en la app en
  **root de la máquina**. El 95 % de los servicios no necesita root.
- Un secreto en `Environment=` lo lee **cualquier usuario** con `systemctl show` o
  `/proc/<pid>/environ`. Va en `EnvironmentFile=` con permisos `0600`.
- Menos servicios corriendo = menos superficie de ataque: `disable --now` + `mask`.

## 🐧 Ojo con los nombres de unit (trampa multi-distro)

| Servicio | Debian / Ubuntu | Arch | Fedora / RHEL |
|---|---|---|---|
| SSH | `ssh.service` | `sshd.service` | `sshd.service` |
| Apache | `apache2.service` | `httpd.service` | `httpd.service` |
| cron | `cron.service` | `cronie.service` | `crond.service` |

Si un comando "no existe", antes de asustarte: `systemctl list-units '*ssh*'`.

---

## ❓ Autoevaluación de la clase

> 5 preguntas para saber si **entendiste lo de hoy**. No hay soluciones — pensalas y, si
> te trabás, abrí la 💡 **Pista**.

### 1️⃣ enable vs start
Habilitaste un servicio con `systemctl enable tienda` y no pasó nada: sigue sin
responder. ¿Por qué? ¿Qué te faltó?
<details><summary>💡 Pista</summary>

Son dos ejes distintos: uno habla del **arranque de la máquina** y el otro del
**ahora**. Hay un flag que hace las dos cosas de una sola vez.
</details>

### 2️⃣ restart vs reload
Cambiaste una línea del `nginx.conf` de producción, en hora pico. ¿`restart` o
`reload`? ¿Y qué pasa si la unit no define `ExecReload=`?
<details><summary>💡 Pista</summary>

Uno **mata y levanta de nuevo** (PID nuevo, conexiones cortadas), el otro le manda una
señal para que **relea la config sin morirse**. Si la unit no dice cómo hacer lo
segundo, systemd te lo va a decir con todas las letras.
</details>

### 3️⃣ daemon-reload
Editaste `/etc/systemd/system/tienda.service`, hiciste `systemctl restart tienda` y
systemd sigue usando la configuración vieja. ¿Qué te falta y por qué?
<details><summary>💡 Pista</summary>

systemd no relee los archivos de unit solo: se guarda una versión **en memoria**. Hay un
comando que le dice "volvé a leer todo desde el disco".
</details>

### 4️⃣ Leer un status
En un `systemctl status` ves `Active: failed (Result: exit-code)` y
`status=203/EXEC`. ¿Qué te está diciendo exactamente ese código, y dónde vas a buscar
el detalle?
<details><summary>💡 Pista</summary>

Los códigos `2xx` son de **systemd**, no de tu programa: significa que ni siquiera
llegó a ejecutarlo. Pensá en la ruta del `ExecStart=` y en los permisos. El detalle
está en el cuaderno: `journalctl -u <unit> -n 50 --no-pager`.
</details>

### 5️⃣ Seguridad + IA
Una IA te escribe un `.service` con `User=root` y
`Environment="DB_PASSWORD=..."`. ¿Qué dos problemas tiene y cómo los arreglás?
<details><summary>💡 Pista</summary>

Uno es de **privilegios** (¿de verdad necesita ser root?) y el otro es una **fuga**:
ese valor lo puede leer cualquiera con `systemctl show` o mirando `/proc/<pid>/environ`.
Hay una directiva hermana que lee los valores desde un archivo con permisos `0600`.
</details>

---

*Material docente. Elaborado con asistencia de IA y revisado por el docente.*
