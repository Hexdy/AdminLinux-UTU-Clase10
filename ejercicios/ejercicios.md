# Ejercicios — Clase 10

Practicá en **tu VM** (los comandos de lectura no necesitan `sudo`) y sobre
`datos/journal_ejemplo.log` y `datos/tienda-api.service`. Soluciones en `soluciones.md`,
pero **intentá primero**.

> 🐧 Si usás Arch o Fedora, acordate: SSH es `sshd`, no `ssh`. Apache es `httpd`, no
> `apache2`. Ante la duda: `systemctl list-units '*ssh*'`.

## Sobre tu máquina (`systemctl`)

1. Averiguá si el servicio de **SSH** de tu distro está corriendo **y** si está
   habilitado al arranque. Son dos preguntas distintas: contestalas con dos comandos
   que devuelvan una sola palabra.
2. Listá **todas** las units de tipo `service` que estén en estado `failed`. ¿Hay
   alguna? Si no hay ninguna, ¿qué te devuelve el comando?
3. Mostrá cuántas units están **habilitadas** al arranque en tu máquina (contalas).
4. ¿En qué **target** arranca tu máquina por defecto? Si fuera un servidor, ¿qué
   target le pondrías y por qué?
5. Averiguá cuánto tardó tu último arranque y cuáles fueron los **3 servicios más
   lentos**.

## Sobre `journalctl`

6. Mostrá las últimas **20** líneas del log del servicio de SSH de tu distro, **sin**
   pager.
7. Mostrá todos los mensajes de prioridad **`err` o peor** del **arranque actual**.
   Después mirá los del arranque **anterior**. ¿Tenés arranques anteriores guardados?
   ¿Qué te dice eso sobre tu journal?
8. Sobre `datos/journal_ejemplo.log`: contá cuántas líneas contienen exactamente
   `Failed with result` y decí qué unidad las generó.

## Sobre un archivo `.service`

9. Abrí `datos/tienda-api.service`. Tiene **5 problemas** (dos de seguridad, tres que
   directamente impiden que el servicio arranque o lo dejan en loop). Encontralos y
   escribí la versión corregida.

## Para pensar (seguridad)

10. Un compañero te dice: "le puse `User=root` porque si no, no me anda". ¿Qué le
    preguntás antes de aceptar eso? Nombrá **dos** directivas de hardening que
    agregarías a esa unit y explicá qué bloquea cada una.

> 🤖 Si te trabás, pedile a la IA que te explique el comando **flag por flag** y
> verificá con una muestra chica antes de confiar en el resultado. Y si te escribe una
> unit, pasala por `systemd-analyze verify` **antes** de habilitarla.
