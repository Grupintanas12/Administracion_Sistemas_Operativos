# Laboratorio de aislamiento de Nginx con AppArmor

## Objetivo

El objetivo de esta práctica es confinar el servidor web Nginx con AppArmor, de forma que pueda servir páginas web con normalidad pero no pueda leer ni modificar archivos críticos del sistema como /etc/shadow, aunque el proceso principal de Nginx se ejecute como root.

Para hacerlo he seguido el proceso que pide la actividad: primero genero el perfil en modo aprendizaje (complain) haciendo peticiones reales al servidor, lo ajusto con aa-genprof, le añado reglas de denegación explícitas y al final lo activo en modo enforce.

## Entorno

He usado la máquina dns2 de las prácticas anteriores, que ya tenía Nginx instalado y sirviendo una página de prueba. Así no toco el servidor maestro, que es el que hace de DNS, DHCP y router.

| Dato | Valor |
| --- | --- |
| Máquina | dns2 (VirtualBox) |
| Sistema operativo | Debian 13 (trixie) |
| IP | 192.168.6.101 (por DHCP desde el maestro) |
| Servidor web | Nginx |
| Acceso desde fuera | http://localhost:8080 en el ordenador anfitrión, con reenvío de puertos en VirtualBox y DNAT con iptables en el maestro |

## 1. Configuración inicial de Nginx

Nginx ya lo había instalado en dns2 en la práctica de NAT y reenvío de puertos:

```
sudo apt update
sudo apt install -y nginx
```

Cambié la página por defecto por una propia en /var/www/html/index.html:

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Prueba nginx</title>
</head>
<body>
  <h1>Hola desde dns2</h1>
  <p>Servidor nginx en 192.168.6.101, accesible a traves del maestro con NAT y reenvio de puertos.</p>
</body>
</html>
```

Y comprobé que funcionaba:

```
sudo systemctl status nginx --no-pager
curl http://localhost
```

El servicio sale `active (running)` y curl devuelve el HTML de la página.

## 2. Creación y ajuste del perfil de AppArmor

### 2.1 Comprobaciones previas

Primero comprobé que AppArmor está activo en el kernel, instalé las herramientas para gestionar perfiles y miré si ya había algún perfil para Nginx:

```
cat /sys/module/apparmor/parameters/enabled
sudo apt install -y apparmor-utils
ls /etc/apparmor.d/ | grep nginx
sudo aa-status | grep nginx
```

El primer comando devuelve `Y`, así que AppArmor está activo. Los dos últimos no devuelven nada: en Debian el paquete de Nginx no trae perfil de AppArmor. Por eso el paso de la guía de poner en complain el perfil `/etc/apparmor.d/usr.sbin.nginx` no se puede hacer tal cual, porque ese fichero no existe. El perfil lo crea desde cero `aa-genprof` en el siguiente paso.

### 2.2 Generación del perfil con aa-genprof

aa-genprof lee los avisos de AppArmor de un fichero de log, y Debian 13 no trae ninguno por defecto (no tiene rsyslog ni auditd). Por eso primero instalé auditd, que guarda los eventos en /var/log/audit/audit.log:

```
sudo apt install -y auditd
```

Para generar el perfil usé dos terminales en dns2. En la primera lancé el asistente:

```
sudo aa-genprof /usr/sbin/nginx
```

aa-genprof crea un perfil vacío para /usr/sbin/nginx, lo pone en modo complain y se queda esperando. En la segunda terminal reinicié Nginx para que arrancara ya bajo ese perfil e hice varias peticiones, una de ellas a una página que no existe para que también se generara un error 404:

```
sudo systemctl restart nginx
curl -s http://localhost/ > /dev/null
curl -s http://localhost/index.html > /dev/null
curl -s http://localhost/noexiste > /dev/null
```

Después volví a la primera terminal y pulsé S para escanear los eventos. Por cada acceso que había hecho Nginx, el asistente me preguntaba si permitirlo (Allow), denegarlo (Deny) o ignorarlo. Estos son los que fueron saliendo y lo que contesté:

| Acceso | Para qué lo usa Nginx | Decisión |
| --- | --- | --- |
| capability dac_override | El proceso principal es root y necesita abrir ficheros que son de otros usuarios (logs de www-data) | Allow |
| capability setuid / setgid | Los procesos trabajadores pasan de root a www-data | Allow |
| /var/www/html/index.html | Leer la página web | Allow, con include web-data |
| /proc/sys/kernel/random/boot_id | Identificador del arranque, lo lee al iniciarse | Allow |
| /run/systemd/userdb/ | Consultar usuarios del sistema | Allow |
| /var/log/nginx/access.log | Escribir el log de accesos | Allow, como /var/log/nginx/* |
| /run/nginx.pid | Guardar el número de proceso para systemd | Allow |
| /etc/nginx/... | Leer su configuración | Allow |
| /etc/passwd, /etc/group, /etc/nsswitch.conf | Encontrar al usuario y grupo www-data | Allow |
| network inet6 stream | Responder a las peticiones por el puerto 80 | Allow, con include nameservice |

Guardé el perfil con S, repetí las peticiones y volví a escanear. Esta segunda vez ya no salió ningún evento nuevo, así que terminé con F. Este es el perfil que generó:

```
abi <abi/4.0>,

include <tunables/global>

/usr/sbin/nginx {
  include <abstractions/base>
  include <abstractions/dovecot-common>
  include <abstractions/nameservice>
  include <abstractions/nis>
  include <abstractions/postfix-common>
  include <abstractions/web-data>

  capability dac_override,

  /etc/nginx/* r,
  /etc/nginx/*/ r,
  /etc/nginx/sites-available/* r,
  /etc/nsswitch.conf r,
  /run/nginx.pid rw,
  /usr/sbin/nginx mr,
  /var/log/nginx/* w,
  owner /etc/group r,
  owner /etc/passwd r,
  owner /proc/sys/kernel/random/boot_id r,
  owner /run/systemd/userdb/ r,

}
```

Al revisarlo vi que tenía tres includes que no pintan nada en un servidor web: dovecot-common y postfix-common (son para servidores de correo) y nis. Se colaron porque en algunas preguntas el asistente proponía por defecto un include en lugar de la regla concreta, y no me fijé. Aunque Nginx funciona igual, esos includes le dan permisos que no necesita, así que en el apartado 4 los quité y puse solo las reglas necesarias.

## 3. Registros en modo complain

Mientras el perfil estaba en modo complain, AppArmor no bloqueó nada pero apuntó en /var/log/audit/audit.log cada acceso que el perfil todavía no permitía. Esos registros llevan `apparmor="ALLOWED"`. Para verlos:

```
sudo grep 'apparmor="ALLOWED"' /var/log/audit/audit.log | grep nginx | head -3
```

Estas son las tres primeras líneas que me salieron:

```
type=AVC msg=audit(1794588127.990:156): apparmor="ALLOWED" operation="open" class="file" profile="/usr/sbin/nginx" name="/etc/nginx/nginx.conf" pid=3108 comm="nginx" requested_mask="r" denied_mask="r" fsuid=0 ouid=0FSUID="root" OUID="root"
type=AVC msg=audit(1794588127.990:157): apparmor="ALLOWED" operation="open" class="file" profile="/usr/sbin/nginx" name="/etc/nsswitch.conf" pid=3108 comm="nginx" requested_mask="r" denied_mask="r" fsuid=0 ouid=0FSUID="root" OUID="root"
type=AVC msg=audit(1794588127.990:158): apparmor="ALLOWED" operation="open" class="file" profile="/usr/sbin/nginx" name="/etc/passwd" pid=3108 comm="nginx" requested_mask="r" denied_mask="r" fsuid=0 ouid=0FSUID="root" OUID="root"
```

Cómo se lee cada línea:

- `apparmor="ALLOWED"`: el acceso se dejó pasar porque el perfil estaba en complain.
- `operation="open"` y `name=`: Nginx abrió ese fichero.
- `requested_mask="r"`: lo abrió para leer.
- `denied_mask="r"`: es el permiso que se habría denegado si el perfil hubiera estado en enforce. Que salga aquí es justo lo que interesa del modo complain: te dice qué se habría bloqueado sin romper el servicio.
- `fsuid=0` y `FSUID="root"`: quien hizo el acceso fue el proceso principal de Nginx, que corre como root.

Lo primero que hace Nginx al arrancar es leer su configuración (nginx.conf) y después nsswitch.conf y passwd, que necesita para encontrar al usuario www-data.

## 4. Perfil final en modo enforce

### 4.1 Ajuste del perfil

Edité el perfil a mano:

```
sudo nano /etc/apparmor.d/usr.sbin.nginx
```

Hice cuatro cambios:

1. Quité los includes dovecot-common, nis y postfix-common, que no tienen nada que ver con un servidor web.
2. Como esos includes eran los que daban a Nginx los permisos para usar el puerto 80 y cambiar de usuario, los añadí como reglas sueltas: `capability net_bind_service`, `capability setgid` y `capability setuid`. Así el perfil da solo esos permisos y no todo lo demás que traían los includes.
3. Cambié las tres reglas de /etc/nginx por una sola, `/etc/nginx/** r`. Los dos asteriscos cubren todo lo que hay dentro de /etc/nginx, subcarpetas incluidas, y solo en lectura.
4. Añadí las reglas de denegación explícitas para /etc/shadow, que es el fichero donde están los hashes de las contraseñas de los usuarios.

El perfil quedó así:

```
abi <abi/4.0>,

include <tunables/global>

/usr/sbin/nginx {
  include <abstractions/base>
  include <abstractions/nameservice>
  include <abstractions/web-data>

  capability dac_override,
  capability net_bind_service,
  capability setgid,
  capability setuid,

  /etc/nginx/** r,
  /etc/nsswitch.conf r,
  /run/nginx.pid rw,
  /usr/sbin/nginx mr,
  /var/log/nginx/* w,
  owner /etc/group r,
  owner /etc/passwd r,
  owner /proc/sys/kernel/random/boot_id r,
  owner /run/systemd/userdb/ r,

  audit deny /etc/shadow r,
  audit deny /etc/shadow w,
}
```

Al principio puse las reglas como `deny /etc/shadow r,` sin el `audit` delante, como venía en la guía. Funcionaban, pero no dejaban rastro en el log (lo explico en el apartado 5.2), así que al final les añadí `audit`.

Para comprobar la sintaxis y cargarlo en el kernel:

```
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```

No devolvió nada, así que el perfil es correcto.

### 4.2 Activación en modo enforce

Con el perfil ya ajustado, lo pasé a modo enforce y reinicié Nginx para que todos sus procesos arrancaran confinados:

```
sudo aa-enforce /usr/sbin/nginx
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager
```

Nginx arrancó sin problemas, así que el perfil no le bloquea nada de lo que necesita para iniciarse. Para confirmar el modo:

```
sudo aa-status | grep -B1 -A3 "enforce mode"
```

```
127 profiles are loaded.
26 profiles are in enforce mode.
...
6 processes have profiles defined.
6 processes are in enforce mode.
   /usr/sbin/cups-browsed (3138)
   /usr/sbin/cupsd (3136)
   /usr/sbin/nginx (3378)
```

El proceso de Nginx aparece en la lista de procesos en modo enforce.

## 5. Comprobación de que Nginx sigue funcionando

### 5.1 La web sigue funcionando

Con el perfil en enforce, hice una petición desde dns2 y busqué en el log si AppArmor había bloqueado algo a Nginx:

```
curl http://localhost
sudo grep DENIED /var/log/audit/audit.log | grep nginx
```

curl devuelve el HTML de mi página y el grep no devuelve nada: en modo enforce AppArmor no ha tenido que bloquear ningún acceso de Nginx, así que el perfil cubre todo lo que necesita.

También lo probé desde fuera, abriendo `http://localhost:8080` en el navegador del ordenador anfitrión. La petición pasa por el reenvío de puertos de VirtualBox y el DNAT del maestro hasta Nginx en dns2, y la página se ve igual que antes de confinarlo.

### 5.2 El bloqueo de /etc/shadow funciona

Para comprobar que las reglas de denegación funcionan de verdad, intenté leer /etc/shadow de dos formas. Uso `wc -l`, que solo cuenta las líneas, para que no salgan los hashes de las contraseñas en las capturas.

Primero como root normal, sin AppArmor:

```
sudo wc -l /etc/shadow
```

```
39 /etc/shadow
```

Después lo mismo, pero con `aa-exec` para que el comando se ejecute bajo el perfil de Nginx, como si fuera el propio Nginx quien intentara leerlo:

```
sudo aa-exec -p /usr/sbin/nginx -- wc -l /etc/shadow
```

```
wc: /etc/shadow: Permission denied
```

Aunque el comando se ejecuta como root, AppArmor lo bloquea. Esto es lo que pasaría si alguien tomara el control de Nginx con una vulnerabilidad e intentara llevarse las contraseñas.

Al buscar el bloqueo en el log al principio no salía nada. Resulta que en AppArmor las reglas `deny` explícitas bloquean en silencio: no se apuntan en el log, porque se supone que ya sabes que quieres bloquear eso. Para que quede registrado hay que poner `audit` delante de la regla. Después de cambiarlo y recargar el perfil con `apparmor_parser -r`, repetí la prueba y ya salió en el log:

```
sudo grep DENIED /var/log/audit/audit.log | grep shadow
```

```
type=AVC msg=audit(1794594064.081:415): apparmor="DENIED" operation="open" class="file" profile="/usr/sbin/nginx" name="/etc/shadow" pid=3526 comm="wc" requested_mask="r" denied_mask="r" fsuid=0 ouid=0FSUID="root" OUID="root"
```

Ahora pone `apparmor="DENIED"` en lugar de ALLOWED: el acceso se ha bloqueado de verdad. El perfil que lo bloquea es el de Nginx (`profile="/usr/sbin/nginx"`), aunque el programa que lo intentó fue `wc` (`comm="wc"`) porque lo lancé con aa-exec. Y `fsuid=0` confirma que lo intentó root.

## 6. Diferencias entre complain y enforce

AppArmor tiene dos modos principales para un perfil:

- **Complain (aprendizaje):** el perfil está cargado pero no bloquea nada. Cuando el programa hace algo que el perfil no permite, AppArmor lo deja pasar y lo apunta en el log. Sirve para ver qué necesita realmente el programa antes de restringirlo. La excepción son las reglas `deny`, que se aplican también en este modo.
- **Enforce (aplicación):** el perfil se aplica de verdad. Todo lo que no está permitido en el perfil se bloquea y además se registra en el log como `DENIED`.

La idea es empezar en complain para no romper el servicio, recoger todo lo que hace Nginx en un uso normal, construir el perfil a partir de eso y cuando está completo pasar a enforce.

## 7. Reflexión

Yo pensaba que a Nginx solo le haría falta leer la página web, pero al hacer el perfil he visto que necesita bastantes más cosas para funcionar:

- Leer su configuración, que está en /etc/nginx.
- Escribir sus logs en /var/log/nginx y el fichero /run/nginx.pid.
- Leer la página web en /var/www.
- Leer /etc/passwd y /etc/group, porque Nginx arranca como root y luego cambia al usuario www-data.
- Algunos permisos de root, como usar el puerto 80 o cambiar de usuario.
- Usar la red para recibir las peticiones y contestarlas.

Lo que más me ha llamado la atención es que Nginx funciona como root, así que sin AppArmor podría leer /etc/shadow, que es donde están las contraseñas. Con el perfil que he hecho, aunque sea root, no puede leerlo, y lo he comprobado con aa-exec.

También me he dado cuenta de que hay que revisar el perfil que crea aa-genprof. A mí se me colaron algunas reglas de programas de correo que no tenían nada que ver con Nginx, y tuve que quitarlas a mano.

Y por último, al principio pensaba que el bloqueo de /etc/shadow no funcionaba porque no salía nada en el log. Luego descubrí que las reglas deny no se apuntan en el log si no les pones audit delante.
