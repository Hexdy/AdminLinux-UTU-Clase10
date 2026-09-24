# Reto de la semana — Clase 10 🏁 (la bandera)

Cerramos la clase como en un trabajo real: **desplegás el entorno con Ansible**, hacés el
**postmortem** de un incidente con las herramientas de systemd, capturás la **bandera** y
**se la decís al profe**.

> 🐧 No es hacking: es la guardia normal de un SRE (leer el journal, entender por qué se
> cayó un servicio y contarlo con números). La bandera es la **recompensa** por hacerlo bien.

> 🏁 **Este reto NO se entrega por Pull Request.** No tenés que subir nada al repo:
> cuando tengas la bandera verificada, **se la decís al profe** (en clase o por el canal
> del curso). Eso es todo.

## El escenario

Sos el SRE de guardia. Anoche `web01` se puso rebelde: hubo servicios que se cayeron,
systemd los reinició hasta que se cansó, y a la mañana alguien pidió un postmortem con
números. Te dejaron el volcado del journal y la foto de las units.

## Pasos

1. **Cloná fuera de Drive**:
   ```bash
   git clone <url> C:\src\AdminLinux-UTU-Clase10
   cd C:\src\AdminLinux-UTU-Clase10\playbooks
   ```
2. **Desplegá la estación con Ansible** (idempotente; simulá primero con `--check`):
   ```bash
   ansible-playbook armar_reto.yml -l local --check --diff
   ansible-playbook armar_reto.yml -l local
   ```
   Esto instala `gawk`, crea `datos/reto/` y te deja tu `MISION.txt`. Corré el playbook
   **dos veces**: la 2ª da `changed=0` (idempotencia).
3. **Hacé el postmortem** sobre `datos/reto/journal_reto.log` (leé `MISION.txt`):
   - **R1** — cuántas líneas contienen **exactamente** `Failed with result`.
   - **R2** — qué **unidad** acumuló más de esas líneas, **sin** el sufijo `.service`.
   - **R3** — cuántos `Scheduled restart job` programó systemd **de esa unidad**.
4. **Armá la bandera** con el formato exacto y **verificala** con Ansible:
   ```bash
   ansible-playbook comprobar_bandera.yml -e "bandera=BANDERA{<R1>-<R2>-<R3>}"
   ```
   Tiene que dar **✅ BANDERA CORRECTA**.
5. **Decile la bandera al profe** — en clase o por el canal del curso. Con eso el reto
   queda entregado. **No hace falta subir nada al repo** ni abrir un Pull Request.

## Las dos trampas (avisadas)

- Hay líneas con `failed with result 'dependency'` en **minúscula**: son units que
  cayeron **por culpa de otra**, no por un fallo propio. **No cuentan** para R1.
  `grep` distingue mayúsculas (si dudás, mirá `grep -i` vs `grep` en una muestra chica).
- Hay líneas `Failed to start <unit>`: ese es el **resumen** que imprime systemd, no el
  resultado. **Tampoco cuentan.**
- Para **R3**: son los `Scheduled restart job` **de la unidad de R2**, no los de todas.

## Cómo se evalúa (autoevaluación)

- [ ] Desplegaste con `armar_reto.yml` y lo corriste **2 veces** (la 2ª da `changed=0`).
- [ ] Leíste la `MISION.txt` que te generó el playbook con **tus** facts.
- [ ] **R1** cuenta solo `Failed with result` exacto (ni minúsculas ni `Failed to start`).
- [ ] **R2** va **sin** el `.service` y es la unidad con **más** fallos (no la primera que viste).
- [ ] **R3** filtra **por esa unidad** antes de contar.
- [ ] `comprobar_bandera.yml` te dio la bandera **correcta** (✅).
- [ ] Le **dijiste la bandera al profe** (en clase o por el canal del curso).

> 🔒 El repo no guarda la bandera en claro, solo su **hash SHA-256**. Así se guardan las
> contraseñas de verdad: nunca el secreto, solo su huella.

> 🤖 Si le pedís el comando a la IA, revisá que el patrón sea el correcto y probalo con
> `head`/`less` sobre una muestra antes de confiar en el número final. Una IA te va a
> tirar `grep -c failed` sin pestañear, y ese número está mal.
