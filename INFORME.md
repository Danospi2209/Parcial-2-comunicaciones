# Informe Técnico: Arquitectura Multi-Contenedor y Análisis del Modelo OSI

## Sección 1: Topología y Flujo de Información

### Diagrama de Arquitectura

                 [ Navegador del Usuario / Cliente ]
                                 |
                          HTTP (Puerto 80:80)
                                 v
                      +--------------------+
                      |       Nginx        |
                      |  (Reverse Proxy)   |
                      +---------+----------+
                                |
       +------------------------+------------------------+
       | (HTTP Interno)         | (WebSockets)           | (HTTP Interno)
       v                        v                        v
+--------------+         +--------------+         +--------------+
|    Joomla    |         |   Jupyter    |         |   Grafana    |
|    (CMS)     |         |    (Lab)     |         |  (Paneles)   |
+------+-------+         +--------------+         +-------+------+
       |                                                  |
       | TCP: 5432                                        | Consultas SQL
       +------------------------+-------------------------+
                                |
                                v
                      +--------------------+
                      |     PostgreSQL     |
                      |   (Base de Datos)  |
                      +--------------------+

### Flujo de Datos y Recolección de Métricas/Logs
1. Entrada de Tráfico: Todas las peticiones HTTP externas ingresan por la interfaz pública de nginx en el puerto 80.
2. Enrutamiento por Prefijos:
   * / se redirige internamente hacia el servicio joomla:80.
   * /jupyter/ se enruta hacia jupyter:8888 manteniendo la conexión bidireccional mediante WebSockets.
   * /grafana/ se enruta hacia grafana:3000.
3. Aprovisionamiento y Métricas: Grafana consulta la base de datos PostgreSQL (database:5432) usando un datasource aprovisionado de forma declarativa para extraer métricas y estadísticas del sistema sin intervención manual.

## Sección 2: Análisis Detallado del Modelo OSI

1. Capa 7 (Aplicación)
   * Cabeceras HTTP inyectadas por Nginx:
       * Host $host: Mantiene el nombre del dominio o host original solicitado por el usuario.
       * X-Real-IP $remote_addr: Transmite la dirección IP real del cliente hacia los contenedores backend.
       * X-Forwarded-For $proxy_add_x_forwarded_for: Conserva la cadena de IPs por las que ha transitado la petición para garantizar la trazabilidad.
       * X-Forwarded-Proto $scheme: Indica si el cliente original utilizó el protocolo HTTP o HTTPS.
   * Mecanismo de HTTP Upgrade (WebSockets): El motor de Jupyter necesita mantener una comunicación bidireccional en tiempo real entre el cliente web y el kernel de Python. Nginx inyecta los encabezados Upgrade $http_upgrade y Connection "Upgrade" para conmutar la conexión HTTP estándar a un túnel de socket persistente.
   * Protocolo de Aplicación de PostgreSQL y Logs: PostgreSQL opera bajo su propio protocolo binario cliente/servidor sobre la Capa 7 para el intercambio de consultas SQL y datos estructurados. Joomla expone logs de acceso y eventos desde su servidor de aplicación.

2. Capa 4 (Transporte)
   * Puertos TCP involucrados:
       * Puerto 80: Publicado hacia el host (0.0.0.0:80->80/tcp) para el tráfico web principal.
       * Puerto 5432: Canal TCP interno entre la base de datos PostgreSQL, Joomla, Grafana y Jupyter.
       * Puerto 8888: Canal TCP interno de Jupyter Lab.
       * Puerto 3000: Canal TCP interno de Grafana.
   * Conexiones Concurrentes y Persistentes: Se emplean mecanismos de TCP keep-alive y reutilización de conexiones (connection pooling) entre el CMS Joomla y PostgreSQL para reducir la latencia del saludo de tres vías (TCP 3-way handshake) en cada consulta.

3. Capa 3 (Red)
   * Segmentación de Redes:
       * frontend_net: Red virtual aislada que intercomunica nginx, joomla, jupyter y grafana.
       * backend_net: Red virtual privada que une exclusivamente joomla, database y grafana. El contenedor database no tiene presencia en frontend_net ni puertos publicados hacia el host exterior.
   * Resolución DNS Interna: Docker administra un servidor DNS interno en la dirección 127.0.0.11 dentro de cada contenedor, lo que permite resolver nombres de servicio lógicos (ej. database, joomla) a sus direcciones IP virtuales asignadas.
   * Reglas de Reenvío y NAT: El kernel del host gestiona la traducción de direcciones de red (NAT) y reglas de iptables para reenviar el tráfico del puerto 80 del host hacia la IP privada de Nginx.

4. Capa 2 (Enlace de Datos)
   * Interfaces Virtuales y Puentes (veth* y br-): Docker crea un puente de red virtual (bridge) en el sistema operativo por cada red configurada (frontend_net y backend_net). Cada contenedor recibe una interfaz de red virtual veth conectada directamente a dicho puente.
   * Resolución ARP Interna: Los contenedores pertenecientes al mismo puente (ej. Nginx y Joomla en frontend_net) utilizan el protocolo ARP (Address Resolution Protocol) para mapear direcciones IP virtuales con direcciones MAC virtuales asociadas a sus interfaces.

## Sección 3: Guía de Verificación y Demostración

1. Verificación del Portal Joomla
   * Acceder a http://localhost desde el navegador.
   * Interactuar con el sitio para generar tráfico HTTP y consultas internas hacia la base de datos PostgreSQL.

2. Verificación de Grafana y Métricas
   * Acceder a http://localhost/grafana/.
   * Abrir el panel de monitoreo preconfigurado (Panel de Monitoreo General) y validar que las gráficas muestran métricas y datos de la base de datos automáticamente sin configuración manual.

3. Verificación de Jupyter Notebook
   * Acceder a http://localhost/jupyter/.
   * Abrir el cuaderno analisis_datos.ipynb dentro de la carpeta de trabajo.
   * Ejecutar las celdas de Python comprobando la correcta ejecución del código.