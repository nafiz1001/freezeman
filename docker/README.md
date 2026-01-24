# Usage

Current working directory is assumed to be the project's root directory.

### Prepare Podman Network for Freezeman

```sh
podman network create freezeman-network
```

## Prepare Database Container

### Prepare Database Extensions

```sh
podman image build --file ./docker/Dockerfile.db-extension --volume freezeman-db-extensions:/extensions --tag freezeman-db-extensions .
podman image rm localhost/freezeman-db-extensions
```

### Prepare The Container

```sh
podman container run --name freezeman-db --publish 5432:5432 --image-volume=ignore --env "POSTGRES_PASSWORD=postgres" -v freezeman-db-data:/var/lib/postgresql/18/docker -v $(pwd)/docker/docker-entrypoint-initdb.d/:/docker-entrypoint-initdb.d/ -v $(pwd)/docker/postgresql.conf:/var/lib/postgresql/18/docker/postgresql.conf:ro -v freezeman-db-extensions:/extensions --network freezeman-network --detach docker.io/postgres:18.1-alpine3.23
# TODO: figure out how the rest would make sense with k8s
source backend/env/bin/activate && python backend/manage.py migrate
podman container stop freezeman-db
# TODO: somehow check if fms database setup done in a k8s
```

Note:
- Initialization files will be executed in sorted name order as defined by the current locale, which defaults to en_US.utf8
- In the case of FreezeMan, you put your .pgsql.gz database dumps there but prepend the filename with a number lower than 2 (e.g. `1-2025-12-17.pgsql.gz`)

## Prepare Backend Container

```sh
podman image build --file ./docker/Dockerfile.backend-prod --tag freezeman-backend .
```

### Prepare Frontend Container

```sh
podman image build --file ./docker/Dockerfile.frontend-prod --tag freezeman-frontend .
```

### Integration

```sh
podman container restart    freezeman-db
podman container run --name freezeman-backend  --network freezeman-network --env 'PG_HOST=freezeman-db' localhost/freezeman-backend
podman container run --name freezeman-frontend --network freezeman-network                              localhost/freezeman-frontend
podman container run --name freezeman-nginx    --network freezeman-network -v $(pwd)/docker/freezeman.nginx.conf:/etc/nginx/nginx.conf:ro -p 8000:80 --rm docker.io/library/nginx:1.29.4-alpine
```
