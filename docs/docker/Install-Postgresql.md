## Compose file

```yaml
version: '3'
services:
  db:
    image: harbor.dockerregistry.com/app/kartoza/postgis:15-3.3
    ports:
      - "5432:5432"
    networks:
      - middleware
    volumes:
      - /data/middleware/postgres/data:/var/lib/postgresql
      - /data/middleware/postgres/dbbackups:/backups
    environment:
      - TZ=Asia/Shanghai
      # If you need to create multiple database you can add coma separated databases eg gis,sys
      - POSTGRES_DB=gis,sys,onemap
      - POSTGRES_USER=postgres
      - POSTGRES_PASS=postgres@123!@#
      - ALLOW_IP_RANGE=0.0.0.0/0
      # Add extensions you need to be enabled by default in the DB. Default are the five specified below
      - POSTGRES_MULTIPLE_EXTENSIONS=postgis,hstore,postgis_topology,postgis_raster,pgrouting
      - RUN_AS_ROOT=true
    restart: on-failure
    healthcheck:
      test: "PGPASSWORD=postgres pg_isready -h 127.0.0.1 -U postgres -d gis"

  dbbackups:
    image: harbor.dockerregistry.com/app/kartoza/pg-backup:15-3.3
    hostname: pg-backups
    networks:
      - middleware
    volumes:
      - /data/middleware/postgres/dbbackups:/backups
    environment:
      - TZ=Asia/Shanghai
      - DUMPPREFIX=PG_db
      - POSTGRES_USER=postgres
      - POSTGRES_PASS=postgres@123!@#
      - POSTGRES_PORT=5432
      - POSTGRES_HOST=db
    restart: on-failure
    depends_on:
      db:
        condition: service_healthy

networks:
  middleware:
    driver: bridge
```





