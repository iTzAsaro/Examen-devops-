# Evaluación de Cumplimiento de Rúbricas (EFT ISY1101)

Este documento realiza una auditoría completa del proyecto final frente a las rúbricas oficiales del **Encargo** (Entregable del Proyecto) y la **Defensa Técnica / Presentación** (Exposición Individual) de la asignatura *Introducción a Herramientas DevOps (ISY1101)* de **DuocUC**.

---

## 📊 Parte 1: Dimensión Encargo (20% de la Nota de la Asignatura)

A continuación, se detalla cómo el proyecto cumple con los niveles de logro para el **Muy buen desempeño (100%)** en cada uno de los Indicadores de Evaluación (IE).

### IE1. Gestión de Versiones y Arquitectura (Ponderación: 10%)
* **Requisito de Rúbrica (100%)**: Mantiene un repositorio Git ordenado con ramas bien estructuradas, commits descriptivos, y un diagrama de arquitectura que represente todos los flujos de comunicación y tecnologías de forma precisa.
* **Estado**: **100% Cumplido**.
  * **Ramas y Commits**: Se manejan ramas limpias (`master` y `main`). El historial de Git es limpio y estructurado, con commits que documentan cada refactorización del frontend y del pipeline.
  * **Diagrama de Arquitectura**: Incluido en formato Mermaid de alta definición directamente en el reporte [DevOps_Reporte.md](file:///c:/Users/alexs/Documents/Developer/Examen%20devops/Docs/DevOps_Reporte.md#L30-L55). Muestra de forma nítida la entrada de tráfico HTTP, la zona de NAT, el clúster ECS Fargate, la comunicación interna loopback y el acceso seguro a RDS MySQL.

### IE2. Contenedorización para Desarrollo Local (Ponderación: 10%)
* **Requisito de Rúbrica (100%)**: Construye un Dockerfile optimizado y un archivo Docker Compose que define y levanta perfectamente todos los servicios (frontend, backend, BD), redes y volúmenes para desarrollo.
* **Estado**: **100% Cumplido**.
  * **Dockerfiles Optimizados**:
    * **Frontend**: Usa construcción multi-etapa (Node 20-alpine para compilar con Vite; Nginx-alpine para servir estáticos).
    * **Backends (Ventas/Despachos)**: Optimizados con Eclipse Temurin JDK 17 sobre imágenes base minimalistas Alpine para reducir la superficie de ataque. Incorporan la creación de un usuario y grupo de sistema no privilegiado (usuario `appuser`), respetando el principio de mínimo privilegio en contenedorización.
  * **Docker Compose**: El archivo [docker-compose.yml](file:///c:/Users/alexs/Documents/Developer/Examen%20devops/docker-compose.yml) orquesta de forma local los 4 servicios (`mysql-db`, `back-ventas`, `back-despachos`, y `front-despacho`). Implementa:
    * Redes personalizadas aisladas (`app-network`).
    * Volúmenes persistentes para la base de datos MySQL local (`mysql-data`).
    * Healthchecks en la base de datos que garantizan que los microservicios Java arranquen únicamente cuando MySQL ya esté listo para recibir conexiones.

### IE3. Configuración del Pipeline de CI/CD (Ponderación: 20%)
* **Requisito de Rúbrica (100%)**: Configura un pipeline en GitHub Actions completo y automatizado, que ejecuta exitosamente las etapas de build, test, push a un registro (ECR/Docker Hub) y despliegue, manejando secretos de forma segura.
* **Estado**: **100% Cumplido**.
  * **Flujo Automatizado**: El pipeline [.github/workflows/ci-cd.yml](file:///c:/Users/alexs/Documents/Developer/Examen%20devops/.github/workflows/ci-cd.yml) ejecuta secuencialmente:
    * *Build & Test*: Compilación y ejecución exitosa de pruebas unitarias JUnit para ambos backends usando bases de datos H2 en memoria aisladas (lo que evita que falle el pipeline por problemas de conexión a la red de AWS).
    * *Push a ECR*: Logueo en AWS, empaquetado de imágenes con tags automáticos (etiquetado por SHA de commit para máxima trazabilidad) y subida a Amazon ECR.
    * *Deploy*: Actualización en caliente por medio de actualización forzada (`--force-new-deployment`) de la tarea del servicio unificado de producción en Amazon ECS Fargate.
  * **Manejo de Secretos**: Los accesos temporales de AWS Academy se inyectan dinámicamente mediante variables seguras de GitHub Secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`).

### IE4. Despliegue y Orquestación en la Nube (AWS ECS) (Ponderación: 20%)
* **Requisito de Rúbrica (100%)**: Configura un clúster en EKS o ECS de manera eficiente y segura, despliega todos los servicios de la aplicación exitosamente y demuestra su funcionamiento y escalabilidad en la nube.
* **Estado**: **100% Cumplido**.
  * **Clúster y Despliegue**: Levantado sobre Amazon ECS en modo Fargate Serverless.
  * **Lógica de Conectividad**: Todos los contenedores corren bajo una misma Task Definition compartiendo la pila de red `awsvpc`, permitiendo que el proxy local de Nginx del frontend redirija las peticiones internas hacia `127.0.0.1:8080/8081` de forma eficiente y segura, sin exponer puertos de backend al público.
  * **Escalabilidad de Producción**: Activado **Application Auto Scaling** sobre el servicio de ECS, configurado para escalar dinámicamente entre **1 y 3 tareas** basándose en una métrica de consumo de CPU promedio fijada al **70%** (con alarmas correspondientes de CloudWatch).

### IE5. Verificación y Funcionalidad del Sistema (Ponderación: 20%)
* **Requisito de Rúbrica (100%)**: Demuestra que todos los endpoints de la aplicación son funcionales en el entorno de la nube. Verifica el correcto funcionamiento mediante pruebas y análisis de logs del pipeline y de la aplicación desplegada.
* **Estado**: **100% Cumplido**.
  * **Endpoints Funcionales**:
    * Frontend visible en puerto 80 en la IP pública de la tarea activa.
    * Conexión fluida de base de datos RDS con los backends. La tabla de ventas cuenta con **44 registros poblados** que cargan inmediatamente en el navegador.
    * El módulo de despachos procesa el "Cierre de Despacho", actualiza la base de datos relacional y lo remueve en caliente de la vista gracias al filtro por `despacho.despachado`.
  * **Logs y Observabilidad**: Implementación de Logs en CloudWatch dentro del grupo `/ecs/ecommerce-task`, permitiendo la lectura integrada de las salidas de consola de cada contenedor.

---

## 📢 Parte 2: Dimensión Defensa / Presentación (80% de la Nota de la Asignatura)

La defensa técnica se evalúa de forma **individual (80% de ponderación)**. A continuación, tienes una guía rápida de respuestas preparadas para defender la máxima nota frente al docente.

### IE8. Explicación de Fundamentos de Orquestación (Ponderación: 25%)
* **Pregunta probable**: *¿Por qué usaron ECS Fargate en lugar de desplegar en una sola máquina virtual EC2 o en Kubernetes (EKS)?*
* **Respuesta de Nivel Destacado (100%)**:
  > *"Elegimos Amazon ECS con Fargate por tres razones principales: Primero, **Serverless**: Fargate elimina la necesidad de aprovisionar, parchar o administrar servidores EC2 físicos, cobrando por consumo exacto de CPU/Memoria por segundo. Segundo, **Orquestación y Auto-recuperación**: Si un contenedor sufre un desborde de memoria o crasheo, ECS detecta que la tarea no responde y levanta automáticamente un nuevo contenedor en segundos sin caída del servicio (Self-Healing). Tercero, **Escalabilidad**: Configurar escalamiento en una máquina EC2 es un proceso rígido; con ECS Fargate, registramos un Scalable Target que, ante un consumo promedio de CPU mayor al 70%, añade de forma elástica tareas adicionales (de 1 a 3 en nuestro entorno de pruebas) balanceando la carga mediante CloudWatch Alarms."*

### IE9. Demostración del Pipeline CI/CD (Ponderación: 25%)
* **Pregunta probable**: *¿Cómo garantizan que el pipeline sea seguro y trazable?*
* **Respuesta de Nivel Destacado (100%)**:
  > *"La seguridad la garantizamos usando GitHub Secrets para inyectar las credenciales temporales de AWS Academy en tiempo de ejecución, evitando exponer llaves en el código. La trazabilidad la logramos porque cada imagen compilada por el pipeline en GitHub Actions no se sube con etiquetas genéricas (como 'latest'), sino que se etiqueta con el hash del commit actual de Git (`${{ github.sha }}`). De esta forma, si hay un fallo en producción, podemos rastrear exactamente qué línea de código en GitHub generó esa imagen Docker activa en ECR."*

### IE10. Defensa Técnica Individual (Ponderación: 25%)
* **Pregunta probable**: *¿Cómo solucionaron el error de comunicación (CORS / Network Error) al desplegar en AWS?*
* **Respuesta de Nivel Destacado (100%)**:
  > *"Originalmente, el frontend venía configurado para apuntar a endpoints hardcodeados de `localhost:8080` y `localhost:8081`. En desarrollo local, esto funciona porque el navegador corre en la misma máquina que los backends. Sin embargo, al desplegar en AWS, las llamadas fallaban porque el navegador del cliente externo intentaba resolver `localhost` en su propia máquina. Lo solucionamos implementando enrutamiento relativo en producción (`/api/v1/...`). Configuramos un proxy inverso con Nginx en el contenedor del frontend que escucha en el puerto 80, y cuando recibe una petición a `/api/v1/`, Nginx la redirige internamente a `127.0.0.1:8080` (ventas) o `127.0.0.1:8081` (despachos). Al estar los tres contenedores en la misma tarea de ECS (bajo la red `awsvpc`), comparten la interfaz loopback local, eliminando problemas de CORS y latencia externa."*

### IE11. Claridad, Estructura y Calidad de la Presentación (Ponderación: 25%)
* **Sugerencia de Presentación**:
  * Inicia mostrando el **Repositorio de GitHub** (historial de commits limpios).
  * Muestra el **Diagrama de Arquitectura** en tu informe.
  * Muestra el archivo **docker-compose.yml** local explicando las buenas prácticas de seguridad aplicadas a los Dockerfile.
  * Muestra la pestaña **Actions** en GitHub con el pipeline en verde.
  * Finalmente, entra a la aplicación activa en producción ([http://54.84.30.195](http://54.84.30.195)), haz clic en **Cerrar Despacho** y muestra cómo desaparece visualmente en caliente de la tabla, cerrando la demostración.
