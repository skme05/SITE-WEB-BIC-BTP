# Déploiement Docker — BIC BTP

## Architecture

| Service    | Image                         | Rôle                                                            |
|------------|-------------------------------|-----------------------------------------------------------------|
| `db`       | `postgres:16-alpine`          | Base de données (volume `postgres_data`)                        |
| `backend`  | `backend/BIC_BTP/Dockerfile`  | Django + Gunicorn. Lance `migrate` et `collectstatic` au démarrage |
| `frontend` | `frontend/Dockerfile`         | Build Vite servi par Nginx, proxy `/api`, `/admin`, `/static` vers Django, sert `/media` |

```
Internet ──443/80──> Nginx du serveur (Certbot) ──> 127.0.0.1:8080 ──> conteneur frontend ──> backend / db
```

Seul le conteneur `frontend` publie un port : `127.0.0.1:8080` (variables `HTTP_BIND` / `HTTP_PORT` dans `.env`).
Il n'est pas joignable depuis l'extérieur, c'est le Nginx installé sur la machine qui gère les ports 80/443 et le certificat.
Le frontend appelle l'API sur la même origine, donc pas de souci de CORS.

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

Vérification depuis le serveur :

```bash
curl http://127.0.0.1:8080/api/health/
```

## Nginx du serveur + Certbot

Les DNS `bic-btp.infinitecorridor.io` et `www.bic-btp.infinitecorridor.io` doivent pointer vers l'IP du serveur, et les ports 80/443 être ouverts.

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

```bash
sudo cp deploy/nginx/bicbtp.conf /etc/nginx/sites-available/bicbtp
```

```bash
sudo ln -s /etc/nginx/sites-available/bicbtp /etc/nginx/sites-enabled/bicbtp
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Génération du certificat (Certbot ajoute le bloc 443 et la redirection HTTP → HTTPS) :

```bash
sudo certbot --nginx -d bic-btp.infinitecorridor.io -d www.bic-btp.infinitecorridor.io --redirect
```

Le renouvellement est automatique (timer systemd de Certbot). Pour le tester :

```bash
sudo certbot renew --dry-run
```

Le site est alors accessible sur `https://bic-btp.infinitecorridor.io/`, l'admin sur `/admin/`, le healthcheck sur `/api/health/`.

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

## Points d'attention

- Le Nginx du serveur transmet `X-Forwarded-Proto: https`, le Nginx du conteneur le relaie tel quel à Django.
- `DJANGO_CSRF_TRUSTED_ORIGINS` doit contenir les URLs `https://...` du site (sinon l'admin refuse la connexion).
- `DJANGO_SECURE_COOKIES=True` uniquement une fois le HTTPS en place.
- Si le port 8080 est déjà pris sur le serveur, changez `HTTP_PORT` dans `.env` **et** le `proxy_pass` de `/etc/nginx/sites-available/bicbtp`.
