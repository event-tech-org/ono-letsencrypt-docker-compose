To integrate these three Docker Compose configurations and ensure Nginx can properly route requests, you need to make sure that Nginx can communicate with the containers running in different Docker Compose projects. Here's how you can achieve that:

1. **Ensure Unique Networks for Each Compose File**:
   Each Docker Compose project can be connected to a common network so they can communicate with each other. Add a common network to each Compose file.

2. **Update Nginx Configuration**:
   Ensure the Nginx configuration points to the correct service names and ports.

3. **Configure Docker Compose Files**:
   Here’s how you can modify each Docker Compose file to include a common network:

### Nginx Compose File

```yaml
version: "3"

services:
  cli:
    build: ./cli
    image: evgeniy-khyst/letsencrypt-docker-compose-cli
    user: ${CURRENT_USER}
    group_add:
      - ${DOCKER_GROUP}
    environment:
      - COMPOSE_PROJECT_NAME
      - CURRENT_USER
      - DOCKER_GROUP
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./:/workdir
    profiles:
      - config

  nginx:
    build: ./nginx
    image: evgeniy-khyst/nginx
    environment:
      - DRY_RUN
    volumes:
      - ./config.json:/letsencrypt-docker-compose/config.json:ro
      - ./nginx-conf/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx-conf/conf.d:/etc/nginx/conf.d:ro
      - ./html:/var/www/html:ro
      - nginx_conf_ssl:/etc/nginx/ssl
      - letsencrypt_certs:/etc/letsencrypt
      - certbot_acme_challenge:/var/www/certbot
    ports:
      - "80:80"
      - "443:443"
    healthcheck:
      test: ["CMD", "nc", "-z", "nginx", "80"]
      interval: 30s
      timeout: 30s
      start_period: 30s
      retries: 10
    restart: unless-stopped
    networks:
      - common_network

  certbot:
    build: ./certbot
    image: evgeniy-khyst/certbot
    environment:
      - DRY_RUN
    volumes:
      - ./config.json:/letsencrypt-docker-compose/config.json:ro
      - letsencrypt_certs:/etc/letsencrypt
      - certbot_acme_challenge:/var/www/certbot
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'test -n "$$(ls -A /etc/letsencrypt/live/)" || test "$$DRY_RUN" == "true" || exit 1',
        ]
      interval: 30s
      timeout: 30s
      start_period: 30s
      retries: 5
    depends_on:
      nginx:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - common_network

  cron:
    build: ./cron
    image: evgeniy-khyst/cron
    environment:
      - COMPOSE_PROJECT_NAME
      - DRY_RUN
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./:/workdir:ro
    depends_on:
      certbot:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - common_network

volumes:
  nginx_conf_ssl:
  letsencrypt_certs:
  certbot_acme_challenge:

networks:
  common_network:
    external: true
```

### Frontend Compose File

```yaml
version: "3"
services:
  web:
    image: eventyay/open-event-frontend
    container_name: opev-frontend
    restart: unless-stopped
    env_file:
      - .env
    environment:
      INJECT_ENV: "true"
      FASTBOOT_DISABLED: "true"
    ports:
      - 4200:4000
    networks:
      - common_network

networks:
  common_network:
    external: true
```

### Backend and Database Compose File

```yaml
version: "3.5"

x-environment-vars: &environment-vars
  POSTGRES_HOST: postgres
  DATABASE_URL: postgresql://open_event_user:opev_pass@postgres:5432/open_event
  REDIS_URL: redis://redis:6379/0
  ADMIN_EMAIL: "@{ADMIN_EMAIL}"
  ADMIN_PASSWORD: "@{ADMIN_PASSWORD}"

x-defaults: &defaults
  image: eventyay/open-event-server:development
  restart: unless-stopped
  env_file:
    - .env
  environment:
    <<: *environment-vars
  depends_on:
    - postgres
    - redis
  volumes:
    - ./static:/data/app/static
    - ./generated:/data/app/generated # Workaround for #6155
  networks:
    - common_network

services:
  postgres:
    image: postgis/postgis:12-3.0-alpine
    container_name: opev-postgres
    restart: unless-stopped
    volumes:
      - pg:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: open_event_user
      POSTGRES_PASSWORD: opev_pass
      POSTGRES_DB: open_event
    ports:
      - 5432:5432
    networks:
      - common_network

  redis:
    image: redis:3-alpine
    container_name: opev-redis
    restart: unless-stopped
    command: redis-server
    volumes:
      - rd:/var/lib/redis/data
    ports:
      - 6379:6379
    networks:
      - common_network

  web:
    <<: *defaults
    container_name: opev-web
    ports:
      - 8080:8080

  celery:
    platform: linux/x86_64
    <<: *defaults
    container_name: opev-celery
    depends_on:
      - web
    environment:
      <<: *environment-vars
      C_FORCE_ROOT: "true"
    command: celery
    networks:
      - common_network

volumes:
  pg:
  rd:

networks:
  common_network:
    external: true
```

### Nginx Configuration

Update the Nginx configuration to use the correct service names and ports. Assuming Nginx and the frontend are on the same host:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name events-qa.goprimetechnologies.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot/events-qa.goprimetechnologies.com;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name events-qa.goprimetechnologies.com;

    include /etc/nginx/ssl/events-qa.goprimetechnologies.com.conf;

    ssl_dhparam /etc/nginx/ssl/ssl-dhparams.pem;

    include /etc/nginx/conf.d/includes/options-ssl-nginx.conf;

    include /etc/nginx/conf.d/includes/hsts.conf;

    resolver 127.0.0.11;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Direct Nginx to proxy requests to the frontend service
        proxy_pass http://opev-frontend:4000;
    }
}
```

### Create the Common Network

Before starting the services, create the common external network:

```bash
docker network create common_network
```

### Start Services

Start each Docker Compose project in the appropriate order:

1. Start the backend and database services:

   ```bash
   docker-compose -f backend-database-compose.yml up -d
   ```

2. Start the frontend services:

   ```bash
   docker-compose -f frontend-compose.yml up -d
   ```

3. Start the Nginx services:

   ```bash
   docker-compose -f nginx-compose.yml up -d
   ```

By following these steps, you ensure that all services are connected to a common network and can communicate with each other. Nginx will be able to route requests to the frontend and backend services correctly.
