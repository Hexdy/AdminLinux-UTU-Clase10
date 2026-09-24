# GitHub Copilot — instrucciones del repo

> Copilot lee este archivo automáticamente. Contexto completo en [`AGENTS.md`](../AGENTS.md).

Repo **educativo** de la Clase 10 de *Administración de Sistemas Linux* (UTU) —
tema **systemd + journalctl (servicios y logs)**. Quien te usa es un **estudiante
principiante**; ayudalo a **aprender**, no solo a que "ande".

Al sugerir código o comandos:

- Respondé en **español rioplatense**, didáctico.
- **Multi-distro**: la VM puede ser Debian/Ubuntu (`apt`), Arch (`pacman`) o Fedora
  (`dnf`); las tres son válidas y las tres usan systemd. Ojo con los nombres de unit:
  `ssh.service` (Debian/Ubuntu) vs `sshd.service` (Arch/Fedora), `apache2` vs `httpd`,
  `cron` vs `cronie`/`crond`. En `playbooks/` el gestor lo elige `ansible_os_family`.
- **Security by design**: nunca `User=root` sin justificación, secretos en
  `Environment=`, `chmod 777` ni secretos versionados. Usuario propio o
  `DynamicUser=yes`, `EnvironmentFile=` en `0600`, y el combo `NoNewPrivileges=true`
  + `ProtectSystem=strict` + `ProtectHome=true` + `PrivateTmp=true`.
- **Nunca** propongas editar `/usr/lib/systemd/system/`: override con `systemctl edit`
  o copia en `/etc/systemd/system/`.
- Recordá el **`daemon-reload`** después de tocar un `.service`, y `systemd-analyze
  verify` **antes** de habilitar nada.
- Explicá qué hace cada comando y cada directiva antes de proponer que lo ejecute.
