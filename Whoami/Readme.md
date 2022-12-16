
## 安裝 Whoami

```bash
> docker-compose -f ./Whoami/docker-compose.yml up -d
```

```docker-compose.yml
version: "3.3"

services:
  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"
    networks:
      - reverse-proxy

networks:
   reverse-proxy:
     external: true
```
