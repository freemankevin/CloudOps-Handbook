## Compose file

```yaml
version: '3.8'
services:
  console:
    image: harbor.dockerregistry.com/app/console:latest
    container_name: console
    ports:
     - '8000:8000'
    volumes:
      - './local_config.yaml:/opt/console/server/local_config.yaml'
    restart: always
```