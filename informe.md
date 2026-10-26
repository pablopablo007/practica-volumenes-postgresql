# Práctica No. 3: Crear volúmenes para persistir una base de datos PostgreSQL

> **Estado del informe:** práctica ejecutada en Ubuntu sobre WSL2 con Docker Engine 29.1.3 e imagen oficial PostgreSQL 16. Las Figuras 2–5 muestran salidas reales de `psql` y Docker capturadas durante la práctica; el registro completo está en [evidencia-practica.txt](evidencia-practica.txt).

## 1. Título

**Persistencia de datos en PostgreSQL con y sin volúmenes Docker**

## 2. Tiempo de duración

**Tiempo de ejecución:** aproximadamente 1 minuto, incluida la descarga de `postgres:16` y las verificaciones. **Entorno:** Ubuntu 22.04 en WSL2, Docker Engine 29.1.3 y PostgreSQL 16.

## 3. Fundamentos

PostgreSQL es un sistema gestor de bases de datos relacional. Organiza la información en bases de datos, tablas y filas, y permite consultarla o modificarla mediante SQL. Para esta práctica se crea una base llamada `test` y, dentro de ella, una tabla `customer` con un identificador, un nombre y un estado. La tabla representa información que debe seguir disponible aunque se reemplace el proceso que ejecuta PostgreSQL.

Docker empaqueta una aplicación y sus dependencias en una imagen. Al iniciar esa imagen, Docker crea un contenedor: un proceso aislado con su propio sistema de archivos, red y procesos. El contenedor facilita iniciar y eliminar instancias de software, pero los archivos escritos en su capa modificable pertenecen al ciclo de vida de ese contenedor. Por eso, al borrar un contenedor, también se pierde lo que solo estaba guardado en esa capa. Volver a iniciar la misma imagen crea otro contenedor limpio; conservar el nombre del contenedor no conserva sus archivos ni las bases de datos.

Un volumen nombrado es un espacio de almacenamiento administrado por Docker cuyo ciclo de vida es independiente del contenedor. Se monta en la ruta donde PostgreSQL guarda sus archivos de datos. El motor escribe allí los archivos de la instancia, incluidas las bases de datos creadas por el usuario. Si se elimina el contenedor y se crea otro usando el mismo volumen y la misma ruta de destino, PostgreSQL encuentra los datos anteriores y vuelve a exponerlos. El volumen no es una copia de seguridad por sí solo: puede borrarse manualmente, dañarse o quedar inaccesible. En una instalación real se requieren copias de seguridad, control de acceso, credenciales seguras y procedimientos de restauración.

La opción `-p` publica un puerto del contenedor en el equipo anfitrión para permitir la conexión de un cliente como DataGrip, TablePlus o `psql`. Por ejemplo, `5432:5432` enlaza el puerto PostgreSQL del anfitrión con el puerto dentro del contenedor. Las variables `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB` configuran la instancia durante su inicialización. La contraseña del ejemplo es solo didáctica y debe cambiarse para cualquier uso real. En ambas partes se utilizan el mismo motor y los mismos datos de prueba; la diferencia experimental es que únicamente la segunda instancia monta el volumen persistente.

La comparación esperada es directa: sin volumen, después de borrar y recrear el contenedor la base `test` no debe encontrarse; con el volumen `pgdata` conservado y montado, la base y el registro insertado deben seguir disponibles. Conviene comprobarlo consultando el catálogo y los registros, y capturar cada resultado. Así se distingue entre la vida del contenedor y la vida de los datos persistentes.

![Esquema de persistencia: los datos sin volumen siguen el ciclo del contenedor; con un volumen nombrado permanecen al recrearlo.](imagenes/persistencia-postgresql.svg)

**Figura 1.** Comparación del almacenamiento de PostgreSQL sin volumen y con volumen Docker.

## 4. Conocimientos previos

- Conceptos básicos de contenedores, imágenes y terminal.
- Manejo básico de Docker CLI y puertos publicados.
- Conexión a PostgreSQL desde un cliente gráfico o `psql`.
- Sentencias SQL para crear bases de datos, tablas, insertar y consultar filas.
- Uso básico de Git y GitHub para publicar el informe.

## 5. Objetivos a alcanzar

- Crear y administrar contenedores PostgreSQL mediante Docker.
- Comprobar la pérdida de datos al eliminar un contenedor sin almacenamiento persistente.
- Crear un volumen nombrado y montarlo en la ruta de datos de PostgreSQL.
- Verificar que la base de datos y sus registros sobreviven al reemplazo del contenedor.
- Explicar la diferencia entre el ciclo de vida del contenedor y el del volumen.

## 6. Equipo necesario

- Computador con Windows, Linux o macOS.
- Docker Desktop o Docker Engine en ejecución.
- Imagen oficial `postgres:16` (Docker la descarga si no está disponible localmente).
- Cliente PostgreSQL: DataGrip, TablePlus, DBeaver o `psql`.
- Conexión local al puerto `5432` libre. Si está ocupado, reemplazar el puerto del anfitrión en `-p` y conectar el cliente a ese nuevo puerto.
- Git y cuenta de GitHub para publicar el informe.

## 7. Material de apoyo

- [Plantilla de informe indicada para la práctica](https://github.com/maguaman2/informe-tendencias/blob/main/informe.md).
- [Docker Docs: volúmenes](https://docs.docker.com/engine/storage/volumes/).
- [Docker Docs: ejecutar contenedores](https://docs.docker.com/engine/containers/run/).
- [Imagen oficial de PostgreSQL en Docker Hub](https://hub.docker.com/_/postgres).
- [Documentación de PostgreSQL](https://www.postgresql.org/docs/).

## 8. Procedimiento

Los comandos están escritos para PowerShell y se ejecutan en una terminal con Docker disponible. Para Linux/macOS, se pueden ejecutar los mismos comandos reemplazando la continuación de línea si se decide dividirlos. En el cliente gráfico, usar **Host** `localhost`, **Puerto** `5432`, **Usuario** `postgres` y la contraseña definida en `POSTGRES_PASSWORD`. Conectarse inicialmente a la base `postgres`; después de crear `test`, cambiar la conexión a `test`.

### Parte 1: PostgreSQL sin volumen

1. Comprobar que Docker responde y crear el contenedor inicial. Reemplazar `CAMBIAR-POR-UNA-CONTRASENA-SEGURA` por una contraseña segura antes de ejecutar. La contraseña usada en la ejecución no se publica.

   ```powershell
   docker version
   docker run -d --name server_db1 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=CAMBIAR-POR-UNA-CONTRASENA-SEGURA -p 127.0.0.1:5432:5432 postgres:16
   docker ps --filter name=server_db1
   ```

2. Con el cliente de base de datos, conectarse a `localhost:5432` con usuario `postgres` y la contraseña configurada. Crear la base de datos:

   ```sql
   CREATE DATABASE test;
   ```

3. Abrir una conexión a la base `test`, crear la tabla e insertar un registro:

   ```sql
   CREATE TABLE customer (
       id SERIAL PRIMARY KEY,
       fullname VARCHAR(120) NOT NULL,
       status VARCHAR(30) NOT NULL
   );

   INSERT INTO customer (fullname, status)
   VALUES ('Ana Pérez', 'activo');

   SELECT id, fullname, status FROM customer;
   ```

4. Detener y eliminar el contenedor, sin usar `-v` ni eliminar volúmenes:

   ```powershell
   docker stop server_db1
   docker rm server_db1
   ```

5. Recrearlo con la misma imagen, nombre y variables, pero sin volumen:

   ```powershell
   docker run -d --name server_db1 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=CAMBIAR-POR-UNA-CONTRASENA-SEGURA -p 127.0.0.1:5432:5432 postgres:16
   docker ps --filter name=server_db1
   ```

6. Esperar a que PostgreSQL termine de iniciar, reconectar el cliente a `localhost:5432` y consultar las bases disponibles:

   ```sql
   SELECT datname FROM pg_database ORDER BY datname;
   ```

   **Resultado observado:** `test` no aparece; el contenedor nuevo inicializó una instancia sin los datos anteriores.

### Parte 2: PostgreSQL con volumen

7. Detener y borrar `server_db1` para liberar el puerto. Crear el volumen nombrado y el contenedor `server_db2` asociado:

   ```powershell
   docker stop server_db1
   docker rm server_db1
   docker volume create pgdata
   docker run -d --name server_db2 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=CAMBIAR-POR-UNA-CONTRASENA-SEGURA -p 127.0.0.1:5432:5432 --mount source=pgdata,target=/var/lib/postgresql/data postgres:16
   docker ps --filter name=server_db2
   docker inspect server_db2
   ```

8. Conectarse al nuevo servidor y crear la misma base, tabla y registro. Si se reutiliza el bloque SQL, la tabla y base deben crearse solo una vez:

   ```sql
   CREATE DATABASE test;
   ```

   Conectado a `test`:

   ```sql
   CREATE TABLE customer (
       id SERIAL PRIMARY KEY,
       fullname VARCHAR(120) NOT NULL,
       status VARCHAR(30) NOT NULL
   );

   INSERT INTO customer (fullname, status)
   VALUES ('Ana Pérez', 'activo');

   SELECT id, fullname, status FROM customer;
   ```

9. Detener y eliminar el contenedor. **No** ejecutar `docker volume rm pgdata` ni `docker volume prune`:

   ```powershell
   docker stop server_db2
   docker rm server_db2
   docker volume ls --filter name=pgdata
   ```

10. Recrear el contenedor con el mismo volumen y volver a conectar el cliente:

    ```powershell
    docker run -d --name server_db2 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=CAMBIAR-POR-UNA-CONTRASENA-SEGURA -p 127.0.0.1:5432:5432 --mount source=pgdata,target=/var/lib/postgresql/data postgres:16
    docker ps --filter name=server_db2
    ```

    Consultar la base `test` y sus filas:

    ```sql
    SELECT id, fullname, status FROM customer;
    ```

    **Resultado observado:** la base `test` y la fila de Ana Pérez permanecieron disponibles después de recrear `server_db2`.

### Evidencias de la ejecución

Las capturas siguientes muestran resultados de consultas ejecutadas en `psql` y verificaciones del Docker CLI. Las imágenes se generaron a partir de la salida real de la terminal, conservada en [evidencia-practica.txt](evidencia-practica.txt).

![Figura 2. Consulta del registro en server_db1 antes de eliminar el contenedor.](evidencias/figura-2-sin-volumen-registro.png)

**Figura 2.** El registro de `customer` existía antes de eliminar `server_db1`.

![Figura 3. Consulta de la base test en server_db1 recreado sin volumen: cero filas.](evidencias/figura-3-sin-volumen-recreado.png)

**Figura 3.** Al recrear `server_db1` sin volumen, PostgreSQL no encontró la base `test`.

![Figura 4. El volumen pgdata se conserva y se monta en el directorio de datos de PostgreSQL.](evidencias/figura-4-volumen-pgdata.png)

**Figura 4.** Docker conservó el volumen `pgdata`, montado en `/var/lib/postgresql/data`.

![Figura 5. Consulta del registro después de recrear server_db2 usando pgdata.](evidencias/figura-5-con-volumen-persistencia.png)

**Figura 5.** Tras recrear `server_db2`, la base `test` y el registro siguieron disponibles.

## 9. Resultados observados

| Escenario | Resultado comprobado | Conclusión |
|---|---|---|
| `server_db1`, sin volumen | Antes de borrarlo, `test.customer` contenía `Ana Pérez`. Después de eliminar y recrear `server_db1`, la consulta a `pg_database` devolvió **0 filas** para `test`. | Los datos guardados solo en la capa del contenedor no sobrevivieron a su eliminación. |
| `server_db2`, con volumen `pgdata` | Después de eliminar y recrear el contenedor con el mismo volumen, la consulta devolvió `id=1`, `fullname=Ana Pérez`, `status=activo`. | El volumen conservó los archivos de PostgreSQL y permitió recuperar la base, la tabla y el registro. |

La práctica se ejecutó con `postgres:16`. Para dejar el servicio local seguro, el contenedor `server_db2` quedó detenido después de la verificación; el volumen `pgdata` sigue creado y los datos persisten. El archivo `evidencia-practica.txt` contiene la salida de las operaciones y consultas.

## 10. Bibliografía

- Docker, Inc. (s. f.). *Volumes*. Docker Documentation. https://docs.docker.com/engine/storage/volumes/
- Docker, Inc. (s. f.). *Running containers*. Docker Documentation. https://docs.docker.com/engine/containers/run/
- Docker Official Images. (s. f.). *postgres*. Docker Hub. https://hub.docker.com/_/postgres
- PostgreSQL Global Development Group. (s. f.). *PostgreSQL Documentation*. https://www.postgresql.org/docs/
- Maguaman2. (s. f.). *Plantilla: informe-tendencias/informe.md*. GitHub. https://github.com/maguaman2/informe-tendencias/blob/main/informe.md
