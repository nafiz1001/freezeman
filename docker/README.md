# Usage

Current working directory is assumed to be the project's root directory.

## Prepare Database Container

```sh
podman image build --file ./docker/Dockerfile --tag freezeman-db .
podman container run --name freezeman-db --publish 5432:5432 --image-volume=ignore --env "POSTGRES_PASSWORD=postgres" -v freezeman-volume:/var/lib/postgresql/18/docker -v $(pwd)/docker/docker-entrypoint-initdb.d/:/docker-entrypoint-initdb.d/ --network freezeman-network --detach freezeman-db
# TODO: figure out how the rest would make sense with k8s
# TODO: somehow check if fms database setup done
source backend/env/bin/activate && python backend/manage.py migrate
podman container stop freezeman-db
```

## Prepare Backend Container

```sh
podman image build --file ./docker/Dockerfile.backend-prod --tag freezeman-backend .
```

### Prepare Frontend Container

```sh
podman image build --file ./docker/Dockerfile.frontend-prod --tag freezeman-frontend .
```

### Prepare Podman Network for Freezeman

```sh
podman network create freezeman-network
```

### Integration

```sh
podman container restart    freezeman-db
podman container run --name freezeman-backend  --network freezeman-network --env 'PG_HOST=freezeman-db' localhost/freezeman-backend
podman container run --name freezeman-frontend --network freezeman-network                              localhost/freezeman-frontend
podman container run --name freezeman-nginx    --network freezeman-network -v $(pwd)/docker/freezeman.nginx.conf:/etc/nginx/nginx.conf:ro -p 8000:80 --rm docker.io/library/nginx:1.29.4-alpine
```
