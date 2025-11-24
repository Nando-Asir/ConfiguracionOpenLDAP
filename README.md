# 📄 Guía de Configuración de Servidor y Cliente LDAP (OpenLDAP en Debian/Ubuntu)

Esta guía detalla la instalación y configuración de un entorno de autenticación centralizada utilizando OpenLDAP.

**Dominio de ejemplo para esta guía:**
* **Dominio Base (Base DN):** `dc=example,dc=com`
* **Servidor LDAP:** `ldap-server.example.com`
* **Cliente LDAP:** `ldap-client.example.com`
* **IP del Servidor:** `[IP_SERVIDOR]` (Ejemplo: `192.168.1.150`)
* **IP del Cliente:** `[IP_CLIENTE]` (Ejemplo: `192.168.1.170`)

---

## 🚀 I. Configuración del Servidor LDAP

Se realiza en la máquina virtual designada como Servidor.

### A. Preparación de Red y Hostname

1.  **Configuración de IP Estática**\
    Modifica el archivo `/etc/network/interfaces` para asignar una IP estática al servidor.

    ```bash
    sudo nano /etc/network/interfaces
    ```

    Añade o modifica las líneas (usando tu interfaz de red, ej: `enp0s3`):

    ```conf
    # /etc/network/interfaces
    auto enp0s3
    iface enp0s3 inet static
    address [IP_SERVIDOR] # Ejemplo: 192.168.1.150
    netmask 255.255.255.0
    gateway [GATEWAY] # Ejemplo: 192.168.1.1
    ```

2.  **Cambio de Hostname**\
    Establece el nombre completo del servidor.

    ```bash
    sudo nano /etc/hostname
    # Dentro del archivo, colocar:
    ldap-server.example.com
    ```

3.  **Modificación del archivo Hosts**\
    Asegúrate de que el servidor se reconozca a sí mismo correctamente.

    ```bash
    sudo nano /etc/hosts
    # Dentro del archivo:
    127.0.1.1    ldap-server.example.com
    [IP_SERVIDOR] ldap-server.example.com ldap-server
    ```

### B. Instalación y Verificación de OpenLDAP

1.  **Actualización del sistema**

    ```bash
    sudo apt update && sudo apt upgrade
    ```

2.  **Instalación del servidor OpenLDAP (`slapd`) y utilidades**\
    Durante la instalación, se te pedirá establecer la **contraseña del administrador** de LDAP.
    
    ```bash
    sudo apt install slapd ldap-utils -y
    ```

3.  **Comprobación del servicio**\
    Verifica que el demonio `slapd` está activo y en ejecución.

    ```bash
    sudo systemctl status slapd.service
    # La salida debe mostrar: Active: active (running)
    ```

4.  **Comprobación del puerto 389**\
    Verifica que el puerto por defecto de LDAP está abierto.

    ```bash
    sudo apt install nmap -y # Instalar nmap si no está disponible
    nmap -p 389 localhost
    # La salida debe mostrar: 389/tcp open ldap 
    ```

5.  **Visualización del contenido inicial (slapcat)**\
    Muestra la estructura base generada por defecto tras la instalación (el Base DN por defecto será el que se configuró en el asistente de instalación).

    ```bash
    sudo slapcat
    # Debe aparecer el Base DN: dn: dc=megainfo212,dc=com (o el configurado)
    ```

### C. Creación de Estructura de Directorio

1.  **Creación del fichero `base.ldif`**\
    Define las Unidades Organizativas (OU) básicas.

    ```bash
    # Crea un directorio y el archivo dentro para mejor organización
    mkdir serverldap && cd serverldap
    nano base.ldif
    ```

    Contenido de `base.ldif`:

    ```ldif
    # base.ldif
    dn: ou=people,dc=example,dc=com
    ou: people
    objectClass: top
    objectClass: organizationalUnit

    dn: ou=group,dc=example,dc=com
    ou: group
    objectClass: top
    objectClass: organizationalUnit
    ```

2.  **Carga del fichero de estructura**\
    Añade las OUs al directorio, se te pedirá la contraseña del administrador de LDAP.

    ```bash
    ldapadd -x -W -D cn=admin,dc=example,dc=com -f base.ldif
    # El comando confirmará: adding new entry "ou=people,dc=example,dc=com"
    ```

3.  **Verificación de la nueva estructura**\
    Busca los DNs recién creados.

    ```bash
    ldapsearch -LL -x -b "dc=example,dc=com" "dn"
    # La salida debe listar: dn: ou=people,dc=example,dc=com y dn: ou=group,dc=example,dc=com
    ```

### D. Creación de Grupo y Usuario POSIX

1.  **Creación del grupo (`sistemas.ldif`)**

    ```bash
    nano sistemas.ldif
    ```

    Contenido de `sistemas.ldif` (Grupo de ejemplo: `sistemas`, GID: `2000`):

    ```ldif
    # sistemas.ldif
    dn: cn=sistemas,ou=group,dc=example,dc=com
    objectClass: top
    objectClass: posixGroup
    cn: sistemas
    gidNumber: 2000
    ```

    Carga el grupo:

    ```bash
    ldapadd -x -W -D cn=admin,dc=example,dc=com -f sistemas.ldif
    ```

2.  **Generación de Contraseña Encriptada**\
    Para el usuario, necesitarás la contraseña encriptada (puedes usar SSHA o MD5).

    ```bash
    # Para obtener el hash SSHA (se usa para el campo userPassword)
    slappasswd
    # Copia el hash resultante (Ej: {SSHA}XXXXXXXX)
    ```

3.  **Creación del usuario (`usuario.ldif`)**\
    Crea el usuario `testuser` con UID y GID coincidentes.

    ```bash
    nano usuario.ldif
    ```

    Contenido de `usuario.ldif` (Usuario de ejemplo: `testuser`, UID/GID: `2000`):

    ```ldif
    # usuario.ldif
    dn: uid=testuser,ou=people,dc=example,dc=com
    objectClass: top
    objectClass: posixAccount
    objectClass: inetOrgPerson
    objectClass: shadowAccount
    uid: testuser
    sn: Generic
    givenName: Test
    cn: Test User
    uidNumber: 2000
    gidNumber: 2000
    userPassword: {SSHA}XXXXXXXX # Reemplazar con el hash generado
    homeDirectory: /home/testuser
    loginShell: /bin/bash
    mail: testuser@gmail.com
    jpegPhoto:
    ```

    Carga el usuario:

    ```bash
    ldapadd -x -W -D cn=admin,dc=example,dc=com -f usuario.ldif
    ```

4.  **Preparación del Directorio Home (Manual)**\
    Este paso asegura que el directorio exista en el servidor antes de la autenticación del cliente.

    ```bash
    # En el servidor LDAP:
    sudo mkdir /home/testuser
    sudo cp -r /etc/skel/.* /home/testuser/
    sudo chown -R 2000:2000 /home/testuser/
    ```

---

## 💻 II. Configuración del Cliente LDAP

Se realiza en la máquina virtual designada como Cliente.

### A. Preparación de Red y Hostname

1.  **Configuración de IP Estática**\
    Configura la red del cliente de forma similar al servidor, pero con su propia IP.

    ```bash
    sudo nano /etc/network/interfaces
    # ...
    address [IP_CLIENTE] # Ejemplo: 192.168.1.170
    # ...
    ```

2.  **Cambio de Hostname**\
    Establece el nombre del cliente.

    ```bash
    sudo nano /etc/hostname
    # Dentro del archivo, colocar:
    ldap-client.example.com
    ```

3.  **Modificación del archivo Hosts**\
    Es **CRUCIAL** que el cliente sepa resolver el nombre del servidor.

    ```bash
    sudo nano /etc/hosts
    # Al final del archivo, añadir:
    [IP_SERVIDOR] ldap-server.example.com
    ```

### B. Instalación y Configuración del Cliente

1.  **Instalación de paquetes de cliente**

    ```bash
    sudo apt update
    sudo apt install libnss-ldap libpam-ldap nslcd -y
    ```
    
    Durante la instalación, el asistente te preguntará:
    * **URI del servidor LDAP:** `ldap://ldap-server.example.com`
    * **Base de búsqueda:** `dc=example,dc=com`

2.  **Configuración del archivo `ldap.conf` (Cliente)**\
    Asegúrate de que el archivo de configuración del cliente tiene el Base DN y el URI correctos.

    ```bash
    sudo nano /etc/ldap/ldap.conf
    # Descomentar/Añadir estas líneas:
    BASE dc=example,dc=com
    URI ldap://ldap-server.example.com
    ```

3.  **Configuración de NSS (`/etc/nsswitch.conf`)**\
    Indica al sistema operativo que debe consultar LDAP para usuarios y grupos.

    ```bash
    sudo nano /etc/nsswitch.conf
    ```

    Modifica las siguientes líneas para añadir `ldap`:

    ```conf
    # /etc/nsswitch.conf
    passwd:         files systemd ldap
    group:          files systemd ldap
    shadow:         files systemd ldap
    gshadow:        files systemd ldap

    hosts:          files myhostname mdns4_minimal [NOTFOUND=return] dns ldap
    networks:       files ldap
    ```

4.  **Reiniciar servicios**\
    Reinicia el servicio de caché de nombres.

    ```bash
    sudo systemctl stop nscd.service
    sudo systemctl restart nscd.service
    # También se puede reiniciar nslcd si se usa este en lugar de nscd
    # sudo systemctl restart nslcd.service
    ```

5.  **Verificación de Acceso a Usuario y Grupo**\
    Comprueba que el sistema ya reconoce al usuario `testuser` y el grupo `sistemas`.

    ```bash
    getent passwd | grep "testuser"
    # Resultado esperado: testuser:x:2000:2000:Test User:/home/testuser:/bin/bash
    getent group | grep "sistemas"
    # Resultado esperado: sistemas:*:2000:
    ```

6.  **Configuración de PAM para Directorio Home**\
    Asegura que, si el directorio no existe, se cree automáticamente al iniciar sesión (o cambiar de usuario) en el cliente.

    ```bash
    sudo nano /etc/pam.d/common-session
    ```

    Añade la siguiente línea (es importante que esté antes de `session optional pam_systemd.so` o al final):

    ```conf
    # /etc/pam.d/common-session
    session required pam_mkhomedir.so
    ```

## ✅ III. Prueba Final

Intenta cambiar de usuario al usuario LDAP.

```bash
# Cambiar a usuario LDAP (se pedirá la contraseña configurada en el servidor)
su testuser
