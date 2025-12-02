# Despliegue del Sistema de Gestión de Proyectos con Autoescalado y Almacenamiento de archivos en S3 en AWS
## Descripción del software
El **Sistema de Seguimiento y Control de Proyectos de Grado para la USFX** es una plataforma web que digitaliza el proceso de titulación: centraliza las versiones de documentos, las observaciones de los docentes, la validación de correcciones y la bitácora de hitos (perfil, borrador, pre-defensa y defensa). Su objetivo es reemplazar el flujo manual en papel por un seguimiento trazable, colaborativo y más rápido entre estudiantes y asesores.

A nivel técnico, el sistema está construido como una aplicación web cliente-servidor con frontend en **Angular** y backend en **NestJS**, apoyado en **PostgreSQL** como base de datos relacional y almacenamiento de documentos en la nube. Para este despliegue se adoptó una arquitectura en AWS orientada a alta disponibilidad y escalabilidad horizontal, basada en los siguientes componentes principales:

* Backend NestJS en EC2, gestionado con PM2 en modo cluster, detrás de un Application Load
* Balancer (ALB) y un Auto Scaling Group (ASG).
* Base de datos PostgreSQL en Amazon RDS, accesible solo desde las instancias del backend.
* Almacenamiento de PDFs en Amazon S3, integrado desde el backend para subir y servir documentos.
* Frontend Angular servido estáticamente desde otra instancia EC2 con Nginx, que consume la API del backend a través del ALB.

Esta estructura en nube permite que el sistema soporte picos de uso, mantenga el backend replicado en varias instancias y aísle los recursos críticos (BD y archivos) dentro de servicios gestionados de AWS.

---
## ARQUITECTURA DE EL DESPLIEGUE EN AWS
![aacd53d98a6997bdf88fd6f1566ff99f.png](:/ba5c6428f9a14e5aaac6bfe829c1dd8b)

## Pasos Realizados en el Despliegue
### 1. Configuración de la base de datos (Amazon RDS – PostgreSQL)

- Se creó una instancia de **Amazon RDS PostgreSQL** que actuará como base de datos central del sistema.
- Para proteger el acceso, se definió un **Security Group exclusivo para RDS**:
  - **Inbound**: solo permite tráfico en el puerto `5432` desde el **Security Group del backend** (el mismo que usarán todas las instancias EC2 que se creen a través del Auto Scaling Group).
  - **Outbound**: por defecto, permite salidas hacia internet (para parches, etc.).
- Los parámetros de conexión (host, puerto, usuario, contraseña, nombre de la base de datos) se guardaron en las variables de entorno del backend (`.env.production`), desactivando sincronización automática de esquemas en producción (`DB_SYNC=false`).

---

### 2. Creación y configuración de la instancia EC2 de backend

1. **Instalación de dependencias**
   - Se lanzó una instancia EC2 base (Ubuntu) para el backend.
   - Se instalaron:
     - Node.js + npm
     - Cliente/driver de PostgreSQL (para herramientas como `psql` si se requieren pruebas)
     - `pm2` como process manager de Node
     - `@nestjs/cli` para tareas de build (opcional en producción)
     - (Inicialmente también Nginx, pero luego se decidió servir el backend directamente en el puerto 80, detrás del ALB).

2. **Despliegue del código**
   - Se clonó el repositorio del backend en `~/proyecto-backend`.
   - Se configuró el archivo `.env.production` con:
     - Conexión a RDS
     - Variables de JWT
     - Configuración de cookies (en producción HTTP: `COOKIE_SECURE=false`, `COOKIE_SAMESITE=lax`)
     - Parámetros de S3.
   - Se ajustó la configuración de **CORS en NestJS** para permitir el origen del frontend (que estará en otro EC2) y habilitar `credentials: true`:
     - `origin`: dominio/IP del frontend
     - `credentials: true`
     - Métodos y headers permitidos.

3. **Build y ejecución con PM2**
   - Se construyó el proyecto:
     ```bash
     cd ~/proyecto-backend
     npm install
     npm run build
     ```
   - Se levantó el backend con `pm2` apuntando al `main.js` compilado en `dist` y escuchando en el puerto `80`:
     ```bash
     sudo pm2 start dist/main.js --name sistema-proyectos-api -i max --env production
     sudo pm2 save
     ```
   - Se configuró `pm2` para arrancar automáticamente tras reinicios:
     ```bash
     sudo pm2 startup systemd -u root --hp /root
     sudo pm2 save
     ```
   - Se validó que el servicio estuviera levantado:
     ```bash
     sudo pm2 status
     curl http://<IP-EC2-backend>/api/auth/login   # Prueba desde la misma instancia
     ```

---

### 3. Creación de la imagen (AMI) del backend

- Una vez que la instancia de backend quedó **funcional y estable**, se creó una **Amazon Machine Image (AMI)** desde esa instancia.
- Esta AMI incluye:
  - Sistema operativo.
  - Código del backend en `~/proyecto-backend`.
  - Dependencias instaladas (`node_modules`).
  - Configuración de `pm2` y su servicio systemd.
- Esta AMI se utilizará como base para todas las instancias que se levanten automáticamente en el Auto Scaling Group.

---

### 4. Creación de la plantilla de lanzamiento (Launch Template)

- Con la AMI creada, se definió una **Launch Template** que especifica:
  - La AMI del backend.
  - Tipo de instancia (por ejemplo, `t3.small`).
  - Security Group del backend.
  - Par de llaves (key pair) para acceso SSH.
  - **User Data** para asegurar que `pm2` y la app se levanten correctamente al iniciar la instancia.  
    Ejemplo de *User Data* (modo texto/script Bash):

    ```bash
    #!/bin/bash
    cd /root/proyecto-backend || cd /home/ubuntu/proyecto-backend

    # En caso de que la imagen ya incluya node_modules y build, solo arrancamos pm2
    sudo pm2 start dist/main.js --name sistema-proyectos-api -i max --env production
    sudo pm2 save

    # Asegurar que el servicio de pm2 esté habilitado (por si alguna instancia nace sin el servicio activo)
    systemctl enable pm2-root || true
    systemctl start pm2-root || true
    ```

- Gracias a esto, cada nueva instancia que el ASG cree arrancará el backend automáticamente en el puerto `80`.

---

### 5. Creación y configuración del Auto Scaling Group (ASG) + Application Load Balancer (ALB)

1. **Creación del ASG**
   - Se creó un **Auto Scaling Group** usando la Launch Template anterior.
   - Configuración clave:
     - Subredes: se seleccionaron todas las subredes públicas/privadas necesarias dentro de la VPC.
     - Capacidad:
       - **Mínima:** 2 instancias
       - **Deseada:** 2 instancias
       - **Máxima:** 4 instancias
     - Estrategia de compra: 100% On-Demand (sin Spot).
   - Se dejó la política de escalado básica por defecto (se podría ajustar más adelante).

2. **Creación del ALB (durante el wizard del ASG)**
   - Se eligió crear un **Application Load Balancer** nuevo, accesible desde internet:
     - Listener: **HTTP 80**
     - Security Group del ALB: permite tráfico entrante en el puerto 80 desde internet (0.0.0.0/0).
   - Se creó un **Target Group** nuevo:
     - Tipo: `Instance`
     - Protocolo: HTTP
     - Puerto: `80` (donde escucha el backend).
     - Health check: HTTP sobre una ruta del backend (por ejemplo `/api/health` o `/api/auth/login` dependiendo de lo disponible).
   - El ASG se configuró para **registrar automáticamente sus instancias** en este Target Group.

3. **Finalización**
   - Se completó el asistente dejando el resto de opciones por defecto hasta crear el ASG.

---

### 6. Verificación del ALB

- Una vez que el ASG lanzó sus primeras instancias:
  - Se comprobó en la consola de EC2 que las instancias estaban en estado **healthy** dentro del Target Group.
  - Se realizaron pruebas con **Postman** apuntando al DNS del ALB:
    ```text
    http://<ALB-DNS>/api/auth/login
    ```
  - Se validó que las llamadas se distribuían correctamente entre las instancias del ASG.
- El **DNS del ALB** se guardó para usarlo como `apiUrl` en la configuración del frontend.

---

### 7. Despliegue del frontend en EC2

1. **Instalación de dependencias**
   - Se lanzó una instancia EC2 separada para el frontend.
   - Se instalaron:
     - Node.js + npm
     - `pm2` (si se desea también gestionar el build con un proceso)
     - `@angular/cli`
     - Nginx como servidor web para los archivos estáticos.

2. **Despliegue del código**
   - Se clonó el repositorio del frontend en `/var/www/proyecto-frontend`.
   - Se configuró `environment.production.ts` para que el `apiUrl` apunte al DNS del ALB:
     ```ts
     export const environment = {
       production: true,
       apiUrl: 'http://<ALB-DNS>/api'
     };
     ```
   - Se construyó la app Angular:
     ```bash
     cd /var/www/proyecto-frontend
     npm install
     ng build --configuration production
     ```

3. **Configuración de Nginx para el frontend**
   - Se configuró un `server` en Nginx escuchando en el puerto `80`, apuntando al build estático:
     ```nginx
     server {
       listen 80;
       server_name <IP-o-DNS-del-frontend>;

       root /var/www/proyecto-frontend/dist/sistema-proyectos-frontend/browser;
       index index.html;

       location / {
         try_files $uri $uri/ /index.html;
       }

       # Cache de assets
       location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
         try_files $uri =404;
         expires 1y;
         add_header Cache-Control "public, immutable";
       }

       location = /index.html {
         add_header Cache-Control "no-cache, no-store, must-revalidate";
       }
     }
     ```
   - Se reinició Nginx y se comprobó que el frontend era accesible mediante el **DNS público de la instancia**.

4. **Pruebas de integración**
   - Desde el navegador se validó:
     - Carga de recursos estáticos del frontend.
     - Llamadas al backend a través del ALB (login, carga de datos, etc.).
     - Envío y recepción correcta de cookies desde el backend.

---

### 8. Configuración de Amazon S3 para almacenamiento de archivos

1. **Creación del bucket S3**
   - Se creó un bucket S3 (por ejemplo en `sa-east-1`) para almacenar documentos y archivos gestionados por el sistema.

2. **Usuario IAM y credenciales**
   - Se creó un usuario IAM con permisos sobre S3 (por ejemplo `AmazonS3FullAccess` o una política más restringida).
   - Se generaron **Access Key ID** y **Secret Access Key**.
   - Estas credenciales se configuraron en el backend mediante variables de entorno:
     ```env
     S3_REGION=sa-east-1
     S3_BUCKET=gestion-proyectos-usfx-grupo-3
     S3_ACCESS_KEY=...
     S3_SECRET_KEY=...
     S3_FORCE_PATH_STYLE=false
     ```

3. **CORS del bucket**
   - Se configuró el **CORS del bucket S3** para permitir que frontend y backend puedan acceder a los archivos, teniendo en cuenta que `pdf.js` carga documentos dentro de la aplicación (no solo descarga).
   - Ejemplo simplificado de configuración CORS:

     ```xml
     <CORSConfiguration>
       <CORSRule>
         <AllowedOrigin>http://<DNS-frontend></AllowedOrigin>
         <AllowedOrigin>http://<ALB-DNS></AllowedOrigin>
         <AllowedMethod>GET</AllowedMethod>
         <AllowedMethod>PUT</AllowedMethod>
         <AllowedMethod>POST</AllowedMethod>
         <AllowedHeader>*</AllowedHeader>
         <ExposeHeader>ETag</ExposeHeader>
       </CORSRule>
     </CORSConfiguration>
     ```

   - Sin esta configuración, `pdf.js` no podría cargar correctamente los documentos dentro del frontend.

---

### 9. Pruebas finales y validación de la arquitectura

- **Autoescalado**
  - Se confirmó que el **ASG lanza nuevas instancias** cuando se ajustan la capacidad deseada o las políticas de escalado.
  - Se verificó que las nuevas instancias pasan a estado **healthy** en el Target Group y empiezan a recibir tráfico a través del ALB.

- **Estado de las instancias**
  - Se accedió por SSH a las instancias nuevas para verificar:
    - Que `pm2` está corriendo el proceso `sistema-proyectos-api`.
    - Que el backend está escuchando correctamente en el puerto `80`.

- **Pruebas con ALB**
  - Se probaron endpoints del backend directamente con el DNS del ALB:
    - Login
    - Carga de recursos
    - Endpoints de negocio principales.
  - Se validó que la respuesta es correcta y que las cookies de autenticación se envían y reciben según lo esperado.

- **Cookies y CORS**
  - Se comprobó que el envío de cookies desde el backend al frontend (dominio distinto) funciona correctamente:
    - Backend configurado con `COOKIE_SECURE=false` y `SameSite=Lax` para entorno HTTP.
    - Angular configurado con `withCredentials: true` en el `HttpClient`.
    - CORS del backend habilitando `credentials: true` y el origen del frontend.
  - Esta fue la parte más delicada, especialmente trabajando aún con **HTTP** (sin HTTPS) y orígenes distintos entre frontend y backend.

---
