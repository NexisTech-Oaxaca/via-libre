---
description: >
  Usar cuando se despliega ViaLibre en AWS, se configura infraestructura cloud (Lambda, ECS,
  Cognito, Location Service, SNS, S3, RDS), se trabaja con variables de entorno de producción,
  o se necesita configurar CI/CD para el monorepo.
---

# Skill: AWS Deployment — ViaLibre

> ⚠️ **Este skill es un placeholder inicial.** La arquitectura y los procedimientos se irán detallando conforme avance el proyecto. Actualizar este archivo cuando se tomen decisiones de infraestructura concretas.

---

## Arquitectura objetivo (propuesta inicial)

```
                        ┌─────────────────────────────────┐
                        │           CloudFront             │
                        │  CDN + caché estático            │
                        └──────────────┬──────────────────┘
                                       │
                ┌──────────────────────┴──────────────────────┐
                │                                             │
    ┌───────────▼──────────┐                    ┌────────────▼───────────┐
    │    S3 Static Hosting │                    │    API Gateway / ALB   │
    │  (Nuxt generado SSG) │                    │  (rutas /api/*)        │
    └──────────────────────┘                    └────────────┬───────────┘
                                                             │
                                                ┌────────────▼───────────┐
                                                │   ECS Fargate          │
                                                │   (Laravel API)        │
                                                └────────────┬───────────┘
                                                             │
                            ┌────────────────────────────────┤
                            │                                │
                ┌───────────▼──────┐             ┌──────────▼──────────┐
                │  Amazon RDS      │             │  ElastiCache Redis   │
                │  (PostgreSQL)    │             │  (cache + queues)    │
                └──────────────────┘             └─────────────────────┘
```

---

## Servicios AWS planeados

| Servicio | Propósito | Estado |
|---|---|---|
| **CloudFront** | CDN para frontend y API | Pendiente configurar |
| **S3** | Hosting frontend (modo SSG) o imágenes de incidentes | Pendiente |
| **ECS Fargate** | Contenedor del backend Laravel | Pendiente |
| **Amazon RDS (PostgreSQL)** | Base de datos principal | Pendiente |
| **ElastiCache (Redis)** | Cache de Laravel + driver de colas | Pendiente |
| **AWS Cognito** | Gestión de usuarios y autenticación | En evaluación |
| **AWS Location Service** | Mapas alternativos y geocoding | En evaluación |
| **AWS SNS** | Notificaciones push y SMS | Pendiente |
| **Lambda** | Procesamiento de eventos async (alternativa a ECS) | En evaluación |
| **SES** | Envío de emails transaccionales | Pendiente |

---

## Variables de entorno por servicio

### Backend (Laravel)

```env
# .env.production — NO commitear al repo
APP_ENV=production
APP_KEY=base64:...

# Base de datos (RDS)
DB_CONNECTION=pgsql
DB_HOST=<rds-endpoint>.rds.amazonaws.com
DB_PORT=5432
DB_DATABASE=vialibre
DB_USERNAME=<usuario>
DB_PASSWORD=<password>

# Cache y colas (ElastiCache)
REDIS_HOST=<elasticache-endpoint>.cache.amazonaws.com
REDIS_PORT=6379
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

# WebSockets (Reverb o Pusher)
REVERB_APP_ID=
REVERB_APP_KEY=
REVERB_APP_SECRET=
REVERB_HOST=
REVERB_PORT=

# AWS (para S3 y SNS)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=vialibre-storage
```

### Frontend (Nuxt)

```env
# .env.production
NUXT_PUBLIC_API_BASE_URL=https://api.vialibre.mx
NUXT_PUBLIC_REVERB_KEY=
NUXT_PUBLIC_REVERB_HOST=
NUXT_PUBLIC_REVERB_PORT=
```

---

## Proceso de despliegue (borrador)

### Backend

```bash
# 1. Build de la imagen Docker
docker build -t vialibre-backend ./backend

# 2. Push a ECR (Elastic Container Registry)
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com
docker tag vialibre-backend:latest <account>.dkr.ecr.us-east-1.amazonaws.com/vialibre-backend:latest
docker push <account>.dkr.ecr.us-east-1.amazonaws.com/vialibre-backend:latest

# 3. Actualizar servicio ECS
aws ecs update-service --cluster vialibre --service backend --force-new-deployment
```

### Frontend

```bash
# 1. Build de Nuxt (modo estático o SSR)
cd frontend
pnpm build         # SSR con Node.js
# o
pnpm generate      # SSG para S3 + CloudFront

# 2. Sincronizar con S3 (modo SSG)
aws s3 sync .output/public s3://vialibre-frontend --delete

# 3. Invalidar caché CloudFront
aws cloudfront create-invalidation --distribution-id <id> --paths "/*"
```

---

## Dockerfile — Backend Laravel

```dockerfile
# backend/Dockerfile
FROM php:8.3-fpm-alpine

WORKDIR /var/www/html

# Dependencias del sistema
RUN apk add --no-cache \
    postgresql-dev \
    nodejs npm

# Extensiones PHP
RUN docker-php-ext-install pdo pdo_pgsql opcache pcntl

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Dependencias PHP
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-scripts

# Código fuente
COPY . .

# Optimizaciones de producción
RUN php artisan config:cache && \
    php artisan route:cache && \
    php artisan view:cache

EXPOSE 8000

CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

---

## TODO — Pendiente de definir

- [ ] Decisión: ECS Fargate vs Lambda (API sin estado vs con estado para WebSockets)
- [ ] Estrategia de WebSockets en producción (Reverb propio vs Pusher/Ably managed)
- [ ] Setup de Cognito vs Sanctum propio para autenticación
- [ ] Configuración de VPC, subnets y security groups
- [ ] CI/CD pipeline (GitHub Actions o AWS CodePipeline)
- [ ] Monitoreo y alertas (CloudWatch, Sentry)
- [ ] Backup automático de RDS
- [ ] Dominio y certificados SSL (Route 53 + ACM)
