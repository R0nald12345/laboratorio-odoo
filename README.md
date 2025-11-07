## Proyecto Odoo local (docker-compose)

Este repositorio contiene una configuración mínima para ejecutar Odoo 17 y Postgres usando Docker Compose.

### 🚀 Comandos Más Importantes (Cheat Sheet)

| Acción | Comando |
|--------|---------|
| **Levantar servicios** | `docker compose up -d` |
| **Bajar servicios** | `docker compose down` |
| **Ver logs en vivo** | `docker compose logs -f odoo` |
| **Abrir shell Odoo** | `docker compose exec odoo bash` |
| **Acceder a Postgres** | `docker compose exec postgres psql -U odoo -d postgres` |
| **Backup DB (sql)** | `docker compose exec -T postgres pg_dump -U odoo postgres > backup.sql` |
| **Backup DB (comprimido)** | `docker compose exec -T postgres pg_dump -U odoo postgres \| gzip > backup.sql.gz` |
| **Backup filestore** | `tar -czf odoo-filestore-$(date +%F).tar.gz odoo-data/filestore` |
| **Ajustar permisos** | `sudo chown -R 1000:1000 odoo-data postgres-data addons` |
| **Reiniciar Odoo** | `docker compose restart odoo` |
| **Ver estado servicios** | `docker compose ps` |
| **Eliminar todo (⚠️ datos)** | `docker compose down -v` |

Resumen rápido
- Servicios: `postgres` (Postgres 15) y `odoo` (Odoo 17).
- Puertos expuestos: Odoo en el host `8069`, Postgres en `5432`.
- Volúmenes locales montados:
  - `./postgres-data` → datos de Postgres
  - `./odoo-data` → datos de Odoo (filestore, sesiones, etc.)
  - `./addons` → carpeta local para addons que se montan en `/mnt/extra-addons` dentro del contenedor Odoo

Archivo docker-compose
El archivo principal es `docker-compose.yml` en la raíz. Define los servicios y volúmenes mencionados arriba.

Acceso
- Interfaz web de Odoo: http://localhost:8069

Comandos útiles (rápidos)
- Levantar en background:
```bash
docker compose up -d
```
- Bajar y eliminar contenedores (preserva volúmenes por defecto):
```bash
docker compose down
```
- Ver logs en tiempo real (ej. Odoo):
```bash
docker compose logs -f odoo
```
- Abrir shell dentro del contenedor Odoo:
```bash
docker compose exec odoo bash
```
- Abrir psql dentro del contenedor Postgres:
```bash
docker compose exec postgres psql -U odoo -d postgres
```

Notas sobre versiones del comando `docker-compose`
- En algunas máquinas el subcomando es `docker-compose` (con guion); en otras (nuevas) `docker compose` (sin guion). Si falla uno, pruebe el otro.

Dónde están los datos
- Base de datos Postgres: `./postgres-data`
- Filestore y configuración de Odoo: `./odoo-data`. Dentro suele haber `filestore/<db_name>/...`.
- Addons personalizados: `./addons` (montado en `/mnt/extra-addons`)

Cómo añadir addons personalizados
1. Copie su módulo dentro de `./addons/mi_modulo`.
2. Ajuste permisos (véase abajo). Un método simple desde el host:
```bash
# ajustar propietario a UID 1000 (usuario típico dentro de contenedores Odoo)
sudo chown -R 1000:1000 ./addons
```
3. Reinicie odoo para que detecte los cambios:
```bash
docker compose restart odoo
```
4. Actualice la lista de apps desde la interfaz o desde la línea de comandos (actualizar módulo base o reiniciar con --dev=reload en casos avanzados).

Permisos y problemas comunes
- Si Odoo no puede leer addons o el filestore, suele ser problema de permisos.
- Los contenedores Odoo oficiales suelen correr procesos con UID 1000. Cambiar el owner en el host a `1000:1000` suele solucionar problemas:
```bash
sudo chown -R 1000:1000 odoo-data postgres-data addons
```
- Si cambió permisos y todavía hay problemas, inspeccione los logs:
```bash
docker compose logs odoo
```

Backups: base de datos
Exportar la base de datos a un SQL (desde el host, volcando la salida a un archivo):
```bash
docker compose exec -T postgres pg_dump -U odoo postgres > backup_postgres_$(date +%F).sql
```
Restaurar a una base vacía (ejemplo simple):
```bash
# parar Odoo para evitar conexiones
docker compose stop odoo
docker compose exec -T postgres psql -U odoo -c "DROP DATABASE IF EXISTS postgres; CREATE DATABASE postgres;"
cat backup_postgres_2025-11-07.sql | docker compose exec -T postgres psql -U odoo -d postgres
```

Backups: filestore (archivos de Odoo)
```bash
tar -czf odoo-filestore-$(date +%F).tar.gz odoo-data/filestore
```
Para restaurar, detenga Odoo, extraiga el tar en `odoo-data/filestore` y ajuste permisos.

Restauración completa (DB + filestore) — resumen
1. Detener Odoo: `docker compose stop odoo`
2. Restaurar base de datos (ver arriba).
3. Restaurar filestore (untar sobre `odoo-data/filestore`).
4. Ajustar permisos: `sudo chown -R 1000:1000 odoo-data`
5. Levantar Odoo: `docker compose up -d odoo`

Copias rápidas y exportación de DB en formato comprimido
```bash
docker compose exec -T postgres pg_dump -U odoo postgres | gzip > backup_postgres_$(date +%F).sql.gz
```

Solución de problemas comunes
- Odoo no arranca y muestra error de conexión a la DB: asegúrese de que `postgres` esté activo y que las credenciales en `docker-compose.yml` coincidan.
- Puerto 8069 ocupado: otro servicio usa el puerto. Cambie el mapeo en `docker-compose.yml` o cierre el servicio en conflicto.
- Errores 500 o problemas al subir archivos: revisar `odoo-data/filestore` y permisos.
- Logs: usar `docker compose logs -f odoo` y `docker compose logs -f postgres`.

Comandos avanzados útiles
- Ejecutar actualización de lista de módulos (modo debug desde el contenedor):
```bash
docker compose exec odoo bash -c "odoo -c /etc/odoo/odoo.conf -d <your_db> -u all --stop-after-init"
```
Sustituya `<your_db>` por el nombre real de la base de datos.

Buenas prácticas
- Mantenga copias regulares de la base de datos y del filestore.
- Use control de versiones (git) para sus addons en `./addons`.
- Si va a desarrollar módulos, usar la opción `--dev=reload` o montar el código con permisos correctos para recarga en caliente puede ahorrar tiempo.

FAQ rápido
- ¿Dónde están las sesiones y archivos temporales? `./odoo-data/sessions` y `./odoo-data/filestore`.
- ¿Cómo crear una base nueva para pruebas? Puede crear una nueva DB desde la interfaz de Odoo o con `createdb`/`psql` en el contenedor Postgres.

Contacto/soporte
Si quieres, puedo:
- Añadir instrucciones para restaurar desde un dump específico que tengas.
- Agregar ejemplos de `docker-compose.override.yml` para desarrollo.
