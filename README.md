# caddy-proxy-reverso-certificados-autofirmados

Guía práctica para implementar un proxy inverso con Caddy sobre Docker en un servidor local que ya corre Pi-hole y Unbound (por ejemplo, sobre OpenMediaVault 7). Esta configuración permite exponer interfaces web de servicios locales mediante HTTPS, usando certificados autofirmados creados por tu propia autoridad certificadora (CA). A diferencia de muchas guías que dependen de Let's Encrypt, esta implementación no requiere acceso externo ni validación pública, ideal para redes cerradas o sin dominio público.

# 🧠 ¿Qué hace esta configuración y por qué te conviene?

Este proyecto te permite acceder de forma segura a servicios como Pi-hole, Nextcloud o Jellyfin desde tu red local, usando HTTPS y sin necesidad de dominios públicos ni puertos abiertos al exterior.

🔐 **HTTPS local con tu propia CA**:
Caddy cifra las conexiones usando certificados autofirmados generados por vos mismo.  
➡️ Así evitás depender de Let's Encrypt o de tener acceso externo, y podés mantener tu red completamente cerrada.

🧭 **Acceso limpio y seguro por nombre de dominio**:
Asignás nombres como https://pihole.local o https://nextcloud.local, y Caddy los redirige automáticamente al servicio correspondiente.  
➡️ Más fácil de recordar, más seguro que IPs o puertos raros, y sin necesidad de configuraciones engorrosas en cada contenedor.

🌐 **Integración simple con redes Docker personalizadas**:
Caddy se comunica con otros contenedores gracias a una red dedicada llamada proxy. Solo él necesita estar en esa red.  
➡️ El resto de tus servicios pueden mantenerse aislados, reduciendo el riesgo y mejorando la organización.

🛠️ **Compatible con Pi-hole + Unbound**:
Este stack está pensado para integrarse directamente con la configuración de DNS recursivo documentada en el repositorio dns-recursivo-pihole-unbound.  
➡️ Si ya usás esa guía, solo tenés que añadir este bloque al final del mismo docker-compose.yml.

📦 **100% local, 0% dependencia externa**:
Todo el stack funciona dentro de tu servidor. No requiere acceso a internet para emitir o renovar certificados.  
➡️ Ideal para entornos cerrados, autohospedados y seguros.

# ⚙️ Configuración paso a paso
## 🧱 1. Preparación del sistema anfitrión
Primero, preparamos el servidor OpenMediaVault para que Caddy pueda operar.
#### ⛓️‍💥 Liberamos los puertos necesarios (80 y 443)
En la interfaz web de OpenMediaVault, ve a Sistema > Entorno de trabajo y cambia los puertos de la interfaz web a unos no estándar (ej. 8080 para HTTP y 8443 para HTTPS).
#### 🛜 Creamos la red docker que vamos a usar
Ejecutas este comando en la consola del host o vía ssh
```bash
docker network create proxy
```

## 📁 2. Preparación de archivos y directorios
Antes de desplegar el stack, es necesario crear las carpetas para los volúmenes y colocar los archivos de configuración.
#### 📂 Crear directorios de volúmenes
```bash
mkdir -p /compose/data/caddy/{config,data,certs}
```
> Estas rutas y nombres de carpetas son solo ejemplos, lo importante es que estén las mínimas requeridas como volumen en el stack (archivo yaml), y que haya una coherencia entre las rutas señaladas en el propio stack y las que hayas creado.
#### 🗃️ Crear el Caddyfile
```bash
sudo nano /compose/data/caddy/config/Caddyfile
```
> Pegá el contenido del archivo homónimo en este repositorio, pero hacé las modificaciones necesarias. Si agregas nuevos servicios a tu servidor, vas a tener que volver a editar este archivo para sumarlos con el mismo comando y respetando el mismo formato que para los dos ejemplos que ya tiene.

## 🔐 Crear los certificados y autoridades de certificación
#### 📂 Navega a la carpeta de certificados
```bash
cd /compose/data/caddy/certs
```
#### 🔑 Crea la clave y el certificado de la CA
Ejecuta los dos comandos siguientes. El segundo te pedirá información; puedes llenarla como prefieras.
```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt
```
#### ⚙️ Crear el archivo de configuración de certificados
```bash
sudo nano /compose/data/caddy/certs/openssl.cnf
```
> Pegá el contenido del archivo homónimo en este repositorio.
#### 🔑 Crea el certificado Wildcard del servidor
```bash
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr -config openssl.cnf
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -extensions v3_req -extfile openssl.cnf
```
>Un certificado Wildcard es especialmente útil porque crea un dominio "comodín" como *.local, que podes usar para cualquier subdominio (o servicio de tu servidor, como pihole.local, nextloud.local, omv.local, etc.).

## 🚀 3. Despliegue y configuración final
#### 🧩 Desplegar el stack desde OMV
1. Ir a Services > Compose > Files
2. Pegar el contenido de docker-compose.yml
> Acá tenés dos opciones: una es copiar a tu stack de Pihole y Unbound el contenido del compose reducido que está en el repositorio (ideal si tenés esos servicios), la otra es crear un nuevo stack para ejecutar el servicio independientemente y crear un nuevo stack con esa configuración.
3. Crear y desplegar el stack
> ⚠️ Nota: *de aquí en adelante, sólo para usuarios de Pihole y Unbound*.
4. Como es la primera vez que levantas Caddy y vas a tener que "bajar" tu contenedor de Pihole y Unbound para hacerlo (con el botón Down en OMV), **seguramente te quedes sin resolución de DNS y no puedas acceder a ninguna web**. Para "arreglarlo" mientras trabajamos, ejecutá los siguientes comandos antes de volver a levantar el contenedor (si intentas levantar el contenedor te va a dar error, nada grave):
``` bash 
sudo chattr -i /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```
5. Levantas el stack.
6. Podes restaurar la configuración anterior.
``` bash
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```
#### ⚙️ Configura Pihole
1. Acceder a la interfaz web de Pi-hole en tu navegador colocando la IP de la máquina host, el signo ":" y el puerto asignado. Por ejemplo: 
``` bash 
192.168.0.20:85
```
2. Andá a Settings > Local DNS Records.
3. Añadí los registros. En "Domain" pones la URL que creaste (por ej. pihole.local) y en IP, la de tu servidor. En todos los servicios que vayas cargando va a ir la misma IP.
> Si agregas nuevos servicios a tu servidor, vas a tener que hacer esto, además de configurar el Caddyfile.
#### (Opcional) Instalar certificados en dispositivos clientes
Esta configuración configura accesos seguros vía https para los servicios, disparando advertencias por parte de los navegadores que no reconocen esa certificación (porque es local y tuya). Si querés evitarlas, podes intentar instalar el certificado del archivo "ca.crt" en tu PC.

