Para entrar al frontend ir a **EC2** -> **Load Balancers** y buscar el **DNS name** del balanceador `alb-ecommerce`. Esto dará un enlace permanente que te llevará a la aplicación:
👉 **URL del balanceador**: [http://alb-ecommerce-647758726.us-east-1.elb.amazonaws.com](http://alb-ecommerce-647758726.us-east-1.elb.amazonaws.com)

---

### 🗄️ Para entrar a la base de datos (RDS MySQL)
Para conectarse directamente a la base de datos en producción desde herramientas como DBeaver, MySQL Workbench o la terminal:

* **Host / Endpoint**: `ecommerce-db.ccix4meuh8yh.us-east-1.rds.amazonaws.com`
* **Puerto**: `3306`
* **Usuario**: `dbuser`
* **Contraseña**: `dbpassword`
* **Bases de Datos lógicas**:
  * `ventas_db` (Almacena transacciones comerciales y compras)
  * `despachos_db` (Almacena logística y cierres de despacho)

*Nota: La base de datos RDS ha sido configurada como "Publicly Accessible" y su Security Group permite conexiones en el puerto 3306 desde cualquier IP (0.0.0.0/0), por lo que puedes conectarte desde cualquier lugar sin modificar reglas de red.*

#### 🔌 Conexión desde AWS CloudShell (Terminal en la Nube)
Dado que AWS CloudShell no incluye el cliente MySQL por defecto, debes instalarlo y luego conectarte con estos comandos:

1. **Instalar el cliente compatible (MariaDB 10.5 Client)**:
   ```bash
   sudo dnf install mariadb105 -y
   ```
2. **Conectarse a la base de datos de Ventas**:
   ```bash
   mysql -h ecommerce-db.ccix4meuh8yh.us-east-1.rds.amazonaws.com -P 3306 -u dbuser -p ventas_db
   ```
   *(Ingresa la contraseña `dbpassword` cuando te sea solicitada).*
3. **Conectarse a la base de datos de Despachos**:
   ```bash
   mysql -h ecommerce-db.ccix4meuh8yh.us-east-1.rds.amazonaws.com -P 3306 -u dbuser -p despachos_db
   ```

4. **Comandos SQL Básicos para la Demostración**:
   Una vez iniciada la sesión en el prompt de MySQL (`mysql>`), puedes ejecutar los siguientes comandos (es **obligatorio** colocar el punto y coma `;` al final de cada comando):
   * **Ver las bases de datos disponibles**:
     ```sql
     SHOW DATABASES;
     ```
   * **Consultar datos de Ventas (`ventas_db`)**:
     ```sql
     USE ventas_db;
     SHOW TABLES;
     DESCRIBE venta;
     SELECT * FROM venta LIMIT 5;
     ```
   * **Consultar datos de Despachos (`despachos_db`)**:
     ```sql
     USE despachos_db;
     SHOW TABLES;
     DESCRIBE despacho;
     SELECT * FROM despacho LIMIT 5;
     ```
   * **Salir del cliente de base de datos**:
     ```sql
     exit;
     ```

---

### ⛓️ Para realizar cambios en el Frontend y ejecutar el Pipeline
Para modificar el frontend y desplegar automáticamente la nueva versión:

1. **Actualizar Credenciales Temporales de AWS Academy**:
   * Entra a la consola de AWS Academy.
   * Haz clic en **AWS Details** -> **Show**.
   * Copia los valores de:
     * `aws_access_key_id`
     * `aws_secret_access_key`
     * `aws_session_token` (Este token es obligatorio para cuentas de estudiante y expira cada 4 horas).
   * Ve a tu repositorio de GitHub -> **Settings** -> **Secrets and variables** -> **Actions**.
   * Actualiza o crea los secretos con los nombres exactos:
     * `AWS_ACCESS_KEY_ID`
     * `AWS_SECRET_ACCESS_KEY`
     * `AWS_SESSION_TOKEN`

2. **Hacer el cambio de código**:
   * Modifica cualquier archivo en la carpeta `front_despacho/src/`.
   * Realiza un commit descriptivo y haz push a la rama `master` o `main`:
     ```bash
     git add .
     git commit -m "feat: actualizar diseño de botones en frontend"
     git push origin main
     ```

3. **Monitorear el despliegue**:
   * Ve a la pestaña **Actions** en tu repositorio de GitHub para ver la ejecución del pipeline en verde.
   * Una vez completado, el pipeline forzará el despliegue en ECS y descargará las nuevas imágenes de ECR con **Zero-Downtime**.

---

### ⚖️ Para demostrar que el Balanceador de Carga y la Escalabilidad funcionan
Durante la presentación, puedes evidenciar el funcionamiento técnico del balanceador y el auto-scaling de la siguiente manera:

1. **Demostración de IP Única y DNS**:
   * Muestra cómo la aplicación carga de manera instantánea utilizando el DNS del ALB: `http://alb-ecommerce-647758726.us-east-1.elb.amazonaws.com`.
   * Explica que las tareas Fargate subyacentes pueden crearse o destruirse (y sus IPs internas/públicas cambiarán), pero el ALB mantiene la misma URL estática para el usuario.

2. **Evidencia de Targets Activos (Target Group)**:
   * Ve a **EC2** -> **Target Groups** -> selecciona `tg-ecommerce-front`.
   * Muestra la pestaña **Targets** (Destinos). Verás que la IP interna de la tarea de ECS Fargate activa está registrada y se reporta en estado **healthy** (saludable) en el puerto 80.

3. **Prueba de Auto Scaling en Vivo (Stress Test)**:
   * **Opción A (Desde PowerShell local)**: Abre 2 o 3 terminales locales en tu máquina y ejecuta:
     ```powershell
     while($true) { Invoke-WebRequest -Uri "http://alb-ecommerce-647758726.us-east-1.elb.amazonaws.com/api/v1/ventas" -UseBasicParsing | Out-Null }
     ```
   * **Opción B (Desde AWS CloudShell - Recomendado)**: Abre AWS CloudShell en tu consola de AWS y ejecuta el siguiente comando en bash para lanzar 10 procesos en paralelo en segundo plano. Esto saturará la CPU del contenedor en segundos debido a la baja latencia interna:
     ```bash
     for i in {1..10}; do (while true; do curl -s -o /dev/null "http://alb-ecommerce-647758726.us-east-1.elb.amazonaws.com/api/v1/ventas"; done &); done
     ```
     *Para detener el stress test en CloudShell, ejecuta:*
     ```bash
     pkill -f curl
     ```
   * En AWS, ve a **CloudWatch** -> **Alarms** y muestra cómo la alarma de CPU alta (70%) se activa y pasa a estado rojo (`ALARM`).
   * Vuelve a **ECS** -> **Clusters** -> `ecommerce-cluster` -> **Tasks** y muestra cómo el clúster levanta automáticamente **2 o 3 tareas concurrentes** para soportar la carga.
   * Muestra la pestaña **Targets** de tu Target Group: el ALB habrá registrado automáticamente las IPs de las nuevas tareas y distribuirá la carga de forma equitativa sin caída del servicio.

4. **Monitoreo de Logs Integrados (CloudWatch)**:
   * **Desde la consola web**: Ve a **CloudWatch** -> **Log groups** -> `ecs/ecommerce-task`. Muestra los flujos de log activos de los tres contenedores (`front-despacho`, `back-ventas`, `back-despachos`) centralizados en un solo lugar.
   * **Desde CloudShell (Recomendado para lucirse en la defensa)**: Puedes ver y seguir los logs en tiempo real directamente en la consola con la CLI de AWS corriendo:
     ```bash
     # Seguir en vivo todos los logs del clúster (Ctrl + C para salir):
     aws logs tail ecs/ecommerce-task --follow --region us-east-1

     # Filtrar logs únicamente del microservicio de Ventas:
     aws logs tail ecs/ecommerce-task --log-stream-name-prefix back-ventas --region us-east-1
     ```
     ```