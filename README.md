# Parcial 2 Práctico: Despliegue Multi-Contenedor, Orquestación y Análisis OSI
**Asignatura:** Comunicaciones  
**Universidad Militar Nueva Granada**  

## Instrucciones de Despliegue (Zero-Touch Deployment)

Para desplegar la infraestructura completa de forma automática, ejecute los siguientes comandos en la raíz del repositorio:

```bash
cp .env.example .env
docker compose up -d
```

## Servicios y URLs de Acceso
* **Joomla CMS:** http://localhost
* **Jupyter Notebook:** http://localhost/jupyter/
* **Grafana Dashboards:** http://localhost/grafana/

## Documentación Técnica
Consulte el archivo [INFORME.md](./INFORME.md) para acceder a la explicación detallada de la arquitectura, flujo de datos y el análisis del Modelo OSI (Capas 2, 3, 4 y 7).