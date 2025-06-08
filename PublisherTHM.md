**Publisher THM**

**Máquina de TryHackme**

**Enumeracion Inicial**

Al hacer un escaneo de puertos estándar con nmap encontramos 2 puertos abiertos:

Puerto 22: SSH

Puerto 80: HTTP


![](/images/Paso.001.png)

Al entrar a se ve una página web con una temática sobre revistas, historias de éxito, tutoriales y opiniones sobre spip. 

SPIP es un software libre de origen francés tipo sistema de gestión de contenidos destinado a la producción de sitios web, orientado a revistas colaborativas en línea e inspirado en los roles de una redacción. Es principalmente utilizado en Francia, en sitios de prensa, en sitios de asociaciones sociales y políticas.

![](/images/Paso.002.png)

En dicha página web no se logra sacar nada de utilidad mas allá de SPIP, el codigo de fuente se ve en orden y la mayoría de links dentro de la página no sirven o llevan a paginas caídas o reales que no se ve que tengan que ver con la maquina.

Al utilizar Gobuster se logra descubrir un directorio llamado /spip el cual lleva a una página web de SPIP.

![](/images/Paso.003.png)

![](/images/Paso.004.png)



**Explotacion de SPIP**

A partir de ahí solo lleva a más publicaciones y artículos sin mucha información de utilidad, al utilizar wappalyzer para ver las tecnologías que utiliza la página web, se puede ver la versión que utiliza SPIP.

![](/images/Paso.005.png)

Al buscar la versión en internet se encuentra un exploit para dicha versión de SPIP 4.2.0 relacionada a RCE o ejecución remota de codigo (CVE:
2023-27372), dicho exploit se encuentra en metasploit asi que ahorramos tiempo preparando el exploit al simplemente utilizar la automatización de metasploit.

Esta vulnerabilidad se debe a un fallo en la forma en que SPIP maneja la serialización de datos en formularios públicos (como los de recuperación de contraseña o contacto). Específicamente, el parámetro oubli puede ser manipulado para incluir código PHP malicioso serializado, que el servidor deserializa e interpreta sin la validación adecuada.

Esto permite a un atacante remoto y sin necesidad de autenticación ejecutar código en el servidor (RCE), con los mismos permisos que el proceso del servidor web, en la mayoría de los casos, esto significa control total del sitio web y, potencialmente, acceso a otros sistemas si hay configuraciones inseguras.

![](/images/Paso.006.png)

El exploit funciono con total normalidad y logrando estar dentro de la maquina lo que permite viajar a la mayoría de directorios y archivos, asi obteniendo la flag del usuario, ahora solo falta encontrar la manera de escalar privilegios y acceder al directorio root que es donde se encuentra la flag que hace falta para completar la máquina.

Antes de seguir con la escalada de privilegios, en la sesión de metapreter se tiene acceso al directorio .ssh del usuario “think” y al contenido que se encuentra dentro, de esta manera se copia el contenido del archivo id\_rsa o la llave privada del usuario a la maquina local atacante para poder acceder al servicio ssh encontrado al principio como el usuario think sin necesidad de una contraseña. 

![](/images/Paso.007.png)


**Post-Explotacion y escalada de privilegios**

Ya dentro de ssh solo falta encontrar la manera de escalar privilegios, el usuario en el que se esta no tiene acceso a comandos o funcionalidades básicas como touch o de lectura y escritura incluso en sus propios directorios.

Al buscar en el sistema por binarios o ejecutables que permitan escalar de privilegios se encontro uno interesante y inusual llamado run\_container, al ejecutarlo aparece un menú sobre contenedores de Docker y la creacion y manejo de estos.

![](/images/Paso.008.png)

![](/images/Paso.009.png)


Al utilizar el comando strings en run\_container encontramos un .sh con el mismo nombre, el comando strings es para inspeccionar el binario y detectar referencias a archivos o comandos internos, lo que permitió descubrir la existencia del script /opt/run\_container.sh.

![](/images/Paso.010.png)

-rwxrwxrwx 1 root root 25 Jun 7 20:10 /opt/run\_container.sh

Este script tiene permisos de lectura y ejecución para todos, entonces simplemente lo que tenemos que hacer es con un editor de texto como nano o vim modificar el codigo para invocar una shell root, ¿no? Aquí entraría el problema.

Se intento modificar o crear archivos en /opt, pero solo se recibian errores de permiso, incluso cuando se intento como usuario think (dueño del home y usuario con el que trabajamos).\
Ejemplos:

- No se puede escribir en /opt/run\_container.sh.
- No se puede crear ni modificar archivos en nuestro propio directorio home (/home/think).
- El intento de ejecutar comandos como chmod o redirecciones a archivos protegidos fallaban.

Esto indicaba que había **r**estricciones de seguridad fuertes en el sistema, no solo por permisos tradicionales de Linux, sino también por una capa adicional: AppArmor.

AppArmor es un módulo de seguridad que puede restringir qué acciones pueden realizar ciertos procesos o usuarios, incluyendo limitaciones para crear, modificar o ejecutar archivos, incluso si los permisos POSIX lo permiten.

Además, notamos que el tipo de shell que tenemos es ash y no bash, lo cual suele limitar funcionalidades y es común en entornos muy restringidos 

Para saltarnos esas restricciones, se tenia que buscar un directorio en el sistema que:

- Tuviera permisos de escritura para todos los usuarios (drwxrwxrwt o similar).
- No estuviera restringido por AppArmor para la creación y ejecución de archivos.
- Permitiera crear scripts o shells para escalar privilegios.

Se uso comandos como:

find / -type d -perm -002 2>/dev/nullfind / -type d -perm -002 2>/dev/null

O simplemente inspeccionamos directorios típicos para archivos temporales, y asi se logro encontrar /dev/shm, que es un sistema de archivos temporal en memoria con permisos:

![](/images/Paso.011.png)

Al probar que efectivamente se podia crear archivo y modificarlos se decidio crear uno llamado shell.sh con el siguiente contenido 

#!/bin/bash

/bin/bash -p

El parámetro -p hace que bash preserve los privilegios efectivos si el binario está marcado con SUID (set user ID), lo que ayuda a mantener privilegios elevados si se usan en binarios especiales, se utilizaron los siguientes comandos para volverlo un ejecutable y se procedió a ejecutar el script:

chmod +x /dev/shm/shell.sh

./dev/shm/shell.sh

Esto permitió abrir una shell bash más potente, sin las limitaciones de ash, con mejor manejo de comandos, más funcionalidades y lo más importante: permitió editar el ejecutable que se encontraba dentro de /opt/run\_container, que recordemos tiene permisos de root.

Al cambiar el codigo por uno que invoca una shell root y luego ejecutarlo cambio la shell por una root, con esto hecho finalmente se logro el acceso al directorio root y a la flag en ella permitiéndo completar la máquina.

![](/images/Paso.012.png)

![](/images/Paso.013.png)
