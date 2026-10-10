1. Verificación del sistema operativo
Objetivo

Verificar que Rocky Linux se ha instalado correctamente.

Comando
Shell
cat /etc/os-release
Explicación

Se comprueba la versión instalada del sistema operativo mediante la consulta del fichero /etc/os-release, verificando que el servidor utiliza Rocky Linux 9.8.

📸 Captura: Resultado de cat /etc/os-release.

2. Identificación de la interfaz de red
Objetivo

Identificar la tarjeta de red que será utilizada para la configuración de red.

Comando
Shell
nmcli device status
Explicación

Se identifica la interfaz de red activa mediante NetworkManager. La interfaz detectada es enp0s3, que será utilizada para la configuración de la dirección IP estática.

📸 Captura: Resultado de nmcli device status.

3. Comprobación de la configuración IP inicial
Objetivo

Verificar la configuración de red existente antes de realizar cambios.

Comando
Shell
ip a
Explicación

Se consulta la configuración de red inicial del sistema observando que la interfaz recibe una dirección IP dinámica mediante DHCP.

📸 Captura: Resultado de ip a.

4. Configuración de IP estática
Objetivo

Asignar la dirección IP requerida para el servidor LDAP.

Comandos
Shell
nmcli connection modify enp0s3 ipv4.addresses 20.0.0.5/24
Shell
nmcli connection modify enp0s3 ipv4.method manual
`
Shell
nmcli connection modify enp0s3 ipv4.gateway 20.0.0.1
Shell
nmcli connection modify enp0s3 ipv4.dns "8.8.8.8 1.1.1.1"
Shell
nmcli connection up enp0s3
Verificación
Shell
ip a
Explicación

Se configura la dirección IP estática 20.0.0.5/24 junto con la puerta de enlace y los servidores DNS necesarios para el acceso a Internet y la comunicación en red.

📸 Captura: Resultado de ip a.

5. Configuración del nombre del servidor
Objetivo

Definir el nombre del equipo.

Comando
Shell
hostnamectl set-hostname ldapserver.planetafp.local
Verificación
Shell
hostname
Explicación

Se configura el nombre del servidor mediante hostnamectl, estableciendo el FQDN ldapserver.planetafp.local.

📸 Captura: Resultado de hostname.

6. Configuración de resolución local
Objetivo

Permitir la resolución local del nombre del servidor.

Edición del fichero
Shell
nano /etc/hosts
Contenido añadido
Plain Text
20.0.0.5 ldapserver.planetafp.local ldapserver
Verificación
Shell
cat /etc/hosts
Explicación

Se añade una entrada en el fichero /etc/hosts para asociar la IP del servidor con su nombre FQDN.

📸 Captura: Resultado de cat /etc/hosts.

7. Verificación de conectividad
Objetivo

Comprobar la conectividad y resolución DNS.

Comandos
Shell
ping -c 4 20.0.0.1
Shell
ping -c 4 8.8.8.8
Shell
ping -c 4 google.com
Explicación

Se realizan pruebas de conectividad para verificar la comunicación con el gateway, el acceso a Internet y la resolución DNS.

📸 Captura: Resultado de ping -c 4 google.com.

8. Actualización del sistema
Objetivo

Actualizar los paquetes del sistema.

Comando
Shell
dnf update -y
Explicación

Se actualiza el sistema operativo para instalar las últimas versiones disponibles de los paquetes.

📸 Captura: No necesaria.

9. Instalación del servidor LDAP
Objetivo

Instalar el servidor LDAP.

Comando
Shell
dnf install 389-ds-base -y
Explicación

En Rocky Linux 9 se utiliza 389 Directory Server como implementación LDAP equivalente funcionalmente a OpenLDAP.

📸 Captura: Instalación finalizada.

10. Inicio del asistente LDAP
Objetivo

Crear y configurar una instancia LDAP.

Comando
Shell
dscreate interactive
Configuración utilizada

Hostname:

Plain Text
ldapserver.planetafp.local

Instancia:

Plain Text
ldapserver

Puerto LDAP:

Plain Text
389

Puerto LDAPS:

Plain Text
636

Directory Manager:

Plain Text
cn=Directory Manager

Sufijo LDAP:

Plain Text
dc=planetafp,dc=local
Explicación

Se ejecuta el asistente interactivo para crear una instancia LDAP mediante 389 Directory Server.

📸 Capturas de las diferentes preguntas del asistente.

11. Verificación del servicio LDAP
Objetivo

Comprobar que el servicio LDAP está en ejecución.

Comando
Shell
systemctl status dirsrv@ldapserver
Explicación

Se verifica que la instancia LDAP se encuentra activa y preparada para procesar solicitudes.

📸 Captura: Estado activo.

12. Verificación del puerto LDAP
Objetivo

Verificar que LDAP escucha en el puerto estándar.

Comando
Shell
ss -tulpn | grep 389
Explicación

Se comprueba que el servicio LDAP escucha correctamente en el puerto 389/TCP.

📸 Captura.

13. Creación de la base LDAP
Objetivo

Crear la entrada raíz del directorio.

Fichero
Shell
nano base.ldif

Contenido:

Shell
ldif no es totalmente compatible. El resaltado de sintaxis se basa en Shell.
dn: dc=planetafp,dc=local
objectClass: top
objectClass: domain
dc: planetafp
 
Verificación
Shell
cat base.ldif
Explicación

Se crea la entrada raíz correspondiente al dominio LDAP.

📸 Captura.

14. Importación de la base LDAP
Comando
Shell
ldapadd -x -D "cn=Directory Manager" -W -f base.ldif
Explicación

Se incorpora la entrada raíz al directorio LDAP.

📸 Captura.

15. Creación de la Unidad Organizativa
Objetivo

Crear la OU usuarios.

Fichero
Shell
nano ou.ldif

Contenido:

Shell
ldif no es totalmente compatible. El resaltado de sintaxis se basa en Shell.
dn: ou=usuarios,dc=planetafp,dc=local
objectClass: organizationalUnit
ou: usuarios
Verificación
Shell
cat ou.ldif
Explicación

Se define una Unidad Organizativa destinada a almacenar usuarios del directorio.

📸 Captura.

16. Importación de la Unidad Organizativa
Comando
Shell
ldapadd -x -D "cn=Directory Manager" -W -f ou.ldif
Explicación

Se incorpora la OU al directorio LDAP.

📸 Captura.

17. Creación del usuario LDAP
Objetivo

Crear un usuario de prueba.

Fichero
Shell
nano usuario.ldif

Contenido:

Shell
ldif no es totalmente compatible. El resaltado de sintaxis se basa en Shell.
dn: cn=Ricard,ou=usuarios,dc=planetafp,dc=local
objectClass: inetOrgPerson
cn: Ricard
sn: Forero
Verificación
Shell
cat usuario.ldif
Explicación

Se define un usuario LDAP utilizando la clase inetOrgPerson.

📸 Captura.

18. Importación del usuario LDAP
Comando
Shell
ldapadd -x -D "cn=Directory Manager" -W -f usuario.ldif
Explicación

Se incorpora el usuario de prueba al directorio LDAP.

📸 Captura con:

Plain Text
adding new entry "cn=Ricard,ou=usuarios,dc=planetafp,dc=local"
19. Verificación final
Objetivo

Validar toda la estructura LDAP creada.

Comando
Shell
ldapsearch -x -D "cn=Directory Manager" -W -b "dc=planetafp,dc=local"
Explicación

Se realiza una consulta autenticada para visualizar todas las entradas almacenadas en el directorio LDAP.

Resultado esperado
Plain Text
dn: dc=planetafp,dc=local
 
dn: ou=usuarios,dc=planetafp,dc=local
 
dn: cn=Ricard,ou=usuarios,dc=planetafp,dc=local
Plain Text
result: 0 Success

📸 Captura final.

