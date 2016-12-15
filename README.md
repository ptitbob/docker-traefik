# docker-traefik
Intégration Docker API / Treafik avec gestion des domaines/sous domaine



```shell
docker-compose -p shipstone -f traefik/docker-compose.traefik.yml up
```

```shell
docker-compose -p shipstone -f front/docker-compose.front1.yml up
```

```shell
docker-compose -p shipstone -f front/docker-compose.front2.yml up
```

```
# shipstone local
fe80::1%lo0	shipstone.local
127.0.0.1	shipstone.local
fe80::1%lo0   web1.shipstone.local
127.0.0.1   web1.shipstone.local
fe80::1%lo0   web2.shipstone.local
127.0.0.1   web2.shipstone.local
```

