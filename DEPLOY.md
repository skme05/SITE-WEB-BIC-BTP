# Déploiement Docker — BIC BTP

## Architecture

| Service    | Image                         | Rôle                                                            |
|------------|-------------------------------|-----------------------------------------------------------------|
| `db`       | `postgres:16-alpine`          | Base de données (volume `postgres_data`)                        |
| `backend`  | `backend/BIC_BTP/Dockerfile`  | Django + Gunicorn. Lance `migrate` et `collectstatic` au démarrage |
| `frontend` | `frontend/Dockerfile`         | Build Vite servi par Nginx, proxy `/api`, `/admin`, `/static` vers Django, sert `/media` |

Seul le port du `frontend` (80 par défaut) est exposé. Le frontend appelle l'API sur la même origine, donc pas de souci de CORS.

## Mise en route

```bash
cp .env.example .env
```

Éditez `.env` : `DJANGO_SECRET_KEY`, `POSTGRES_PASSWORD`, `DJANGO_ALLOWED_HOSTS`, les identifiants email, etc.
Pour générer une clé secrète :

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(64))"
```

```bash
docker compose up -d --build
```

```bash
docker compose exec backend python manage.py createsuperuser
```

Le site est accessible sur `http://<serveur>/`, l'admin sur `/admin/`, le healthcheck sur `/api/health/`.

## Commandes utiles

```bash
docker compose logs -f backend
```

```bash
git pull && docker compose up -d --build
```

```bash
docker compose exec db pg_dump -U bicbtp bicbtp > backup.sql
```

## HTTPS

Placez un reverse proxy TLS devant le conteneur `frontend` (Caddy, Traefik, Nginx + Certbot, ou le proxy de votre hébergeur).
Django lit l'en-tête `X-Forwarded-Proto` pour savoir que la requête est en HTTPS.
Pensez à mettre les URLs `https://...` dans `DJANGO_CSRF_TRUSTED_ORIGINS`.
