# Control-de-Acceso-Obligatorio-confinamiento-Nginx-con-AppArmor

Control de Acceso Obligatorio: confinamiento Nginx con AppArmor


Instalamos nginx y comprobamos que este encendido
1. Actualizar los repositorios:
sudo apt update
2. Instalar Nginx:
sudo apt install nginx -y
3. Iniciar el servicio y configurarlo para que arranque automáticamente:
sudo systemctl enable --now nginx
4. Comprobar su estado:
sudo systemctl status nginx



5. Comprobar que responde localmente:
curl -I http://localhost
La respuesta debería incluir:


Paso 2. Comprobar AppArmor
Primero vamos a comprobar que AppArmor está instalado y activo en Ubuntu Server.
1. Comprobar el estado del servicio:
sudo systemctl status apparmor --no-pager
Debería aparecer:
2. Comprobar que AppArmor está habilitado en el kernel:
sudo aa-status
Deberías ver información sobre:
Módulo AppArmor cargado.
Número de perfiles cargados.
Perfiles en modo enforce.
Perfiles en modo complain.
3. Comprobar que tienes las herramientas necesarias:
which aa-genprof
which aa-enforce
which aa-complain
Si no aparecen las rutas, instala las herramientas:
sudo apt install apparmor apparmor-utils -y

4. Comprobar la versión:
apparmor_parser --version



Paso 3. Generar el perfil de AppArmor para Nginx
1. Comprueba la ruta del ejecutable de Nginx:
which nginx
Normalmente devolverá:

2. Comprueba si ya existe un perfil:
sudo ls /etc/apparmor.d/ | grep nginx
Si no aparece ningún resultado, podemos generar uno desde cero.
3. Inicia el generador de perfiles:
sudo aa-genprof /usr/sbin/nginx
Se abrirá el asistente interactivo de AppArmor.
El procedimiento será:
aa-genprof crea un perfil inicial para Nginx.
El perfil se coloca en modo complain.
Se generan eventos mientras Nginx realiza operaciones.
El asistente analiza los eventos y propone permisos.
¿Qué significa el modo complain?
Registra los accesos que no están contemplados en el perfil.
Permite esos accesos en lugar de bloquearlos.
Nos ayuda a descubrir qué permisos necesita Nginx.
4. Cuando el asistente esté abierto:
Déjalo abierto y, desde otra terminal SSH o consola, ejecuta:
sudo systemctl restart nginx
Nginx no podía acceder a /etc/nginx/nginx.conf porque su perfil de AppArmor estaba activo en modo enforce y bloqueaba el acceso. 
Después genera actividad HTTP:
curl http://localhost

También puedes ejecutar varias peticiones:
for i in {1..10}; do curl -s http://localhost > /dev/null; done
5. Vuelve a la terminal donde está aa-genprof.
Pulsa S para analizar los registros y sigue las opciones que vaya mostrando el asistente. No aceptes permisos indiscriminadamente: revisaremos las propuestas.
6. Consulta los eventos de AppArmor:
sudo journalctl -k --since "10 minutes ago" | grep -i apparmor
También puedes consultar:
sudo dmesg | grep -i apparmor
7. Guarda las evidencias.
Para conservar los registros en un archivo:
sudo journalctl -k --since "10 minutes ago" | grep -i apparmor > ~/eventos-apparmor-nginx.txt
Paso 4. Creación y configuración del perfil de AppArmor para Nginx
Primero comprobamos que AppArmor estaba instalado y activo en Ubuntu Server mediante:
sudo aa-status
Detectamos que Nginx ya disponía de un perfil de AppArmor en modo enforce, lo que provocaba que el servicio no pudiera acceder a /etc/nginx/nginx.conf y fallara al reiniciarse.
Para solucionar el problema temporalmente, cambiamos el perfil a modo aprendizaje:
sudo aa-complain /usr/sbin/nginx
En este modo, AppArmor registra los accesos no contemplados en el perfil, pero no los bloquea.
A continuación, reiniciamos Nginx y generamos peticiones HTTP para registrar su actividad:
sudo systemctl restart nginx
curl http://localhost
Consultamos los eventos registrados:
sudo journalctl -k -n 30 | grep -i apparmor
4.1. Análisis de los registros
Los registros mostraron los accesos que Nginx necesitaba para funcionar correctamente:
Recurso
Permiso
Finalidad
/etc/nginx/nginx.conf
Lectura
Configuración principal
/etc/nginx/modules-enabled/
Lectura
Consultar módulos
/etc/nginx/mime.types
Lectura
Identificar tipos de archivos
/etc/nginx/conf.d/
Lectura
Consultar configuraciones adicionales
/etc/nginx/sites-enabled/
Lectura
Consultar sitios habilitados
/etc/nginx/sites-available/default
Lectura
Configuración del sitio predeterminado
/etc/nsswitch.conf
Lectura
Consultar resolución de usuarios y nombres
/etc/passwd
Lectura
Consultar usuarios
/etc/group
Lectura
Consultar grupos
/run/nginx.pid
Lectura y escritura
Gestionar el proceso
/var/www/html/
Lectura
Servir contenido web

4.2. Ajuste del perfil
Para analizar los registros y generar las reglas necesarias ejecutamos:
sudo aa-logprof
El asistente interactivo presentó diferentes accesos detectados y seleccionamos los permisos necesarios para el funcionamiento de Nginx.
Para el contenido web seleccionamos la abstracción:
#include <abstractions/web-data>
Esta abstracción permite reutilizar reglas predefinidas relacionadas con el acceso a contenido web.
Finalmente, guardamos los cambios seleccionando la opción Save Changes.
Paso 5. Activar el modo enforce
Ahora vamos a aplicar las restricciones de AppArmor.
Ejecuta:
sudo aa-enforce /usr/sbin/nginx
Comprueba el resultado:
sudo aa-status | grep nginx


5.1. Ajuste del perfil
Modo
Funcionamiento
Complain
Registra los accesos no permitidos, pero no los bloquea.
Enforce
Bloquea los accesos que no están autorizados por el perfil.

Conclusión: El modo complain permite identificar los recursos que necesita Nginx para funcionar. Después, con aa-logprof, podemos ajustar el perfil y finalmente aplicar las restricciones mediante enforce.
Paso 6. Comprobar Nginx con AppArmor en modo enforce
Vamos a verificar que el servidor web sigue funcionando después de aplicar el confinamiento.
1. Reiniciar Nginx:
sudo systemctl restart nginx
2. Comprobar su estado:
sudo systemctl status nginx --no-pager
Debe aparecer:

3. Comprobar que responde:
curl -I http://localhost
Debe devolver:
4. Confirmar que AppArmor sigue en enforce:
sudo aa-status
Busca /usr/sbin/nginx dentro de los perfiles en modo enforce.

