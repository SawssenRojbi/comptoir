# Comptoir

Comptoir est une petite application e-commerce B2B de fournitures professionnelles (outillage, quincaillerie, livres techniques, bureau).
Le dépôt contient deux applications :
- `api/` : API REST Laravel 12 (PHP 8.3), base PostgreSQL 16, tests Pest ;
- `web/` : front Next.js (App Router, TypeScript), tests Vitest et Playwright.
Le front appelle l'API en JSON ; les montants circulent en centimes.

## Prérequis

- Git
- PHP 8.3 avec Composer et l'extension PCOV (couverture de code)
- Node.js 20 ou plus récent (npm inclus)
- Docker (avec Docker Compose) **ou** PostgreSQL 16 installé localement, avec un superutilisateur `postgres`
- `jq`

## Installation

### 1. Base de données

Avec Docker :

```bash
docker compose up -d
```

Le conteneur crée le rôle `comptoir` (mot de passe `comptoir`) et les bases `comptoir` (développement), `comptoir_test` (tests) et `comptoir_e2e` (tests de bout en bout en CI). Le port exposé est `5432` ; s'il est déjà pris, lancez `COMPTOIR_DB_PORT=5433 docker compose up -d` et reportez le port dans `DB_PORT` (`api/.env`).

Sans Docker, exécutez le même script avec le superutilisateur de votre PostgreSQL local :

```bash
psql -U postgres -f docker/postgres/init/01-init.sql
```

Pour exécuter un script SQL d'administration dans le conteneur (par exemple la création d'un rôle en lecture seule) :

```bash
docker compose exec -T postgres psql -U postgres -d comptoir < chemin/vers/create-readonly-role.sql
```

### 2. API

```bash
cd api
composer install
cp .env.example .env
php artisan key:generate
php artisan migrate --seed
php artisan serve
```

L'API répond sur <http://127.0.0.1:8000> (`GET /` renvoie `{"name":"Comptoir API","status":"ok"}`, le catalogue est sur `/api/products`).

Comptes de démonstration (mot de passe `password`) : `karim@atelier-nord.example.test`, `lina.mansour@example.test`, `einkauf@bau-werkzeuge.example.test`, `admin@comptoir.example.test`.

### 3. Front

Dans un second terminal :

```bash
cd web
npm ci
cp .env.example .env.local
npm run dev
```

Le site répond sur <http://localhost:3000>.

## Commandes utiles

### Tests et qualité

| Commande | Dossier | Rôle |
|---|---|---|
| `php artisan test` | `api/` | Tests Pest (base `comptoir_test`) |
| `vendor/bin/pest --testsuite=Unit` | `api/` | Tests unitaires seuls (sans base de données) |
| `./vendor/bin/pint --test` | `api/` | Vérification du formatage PHP |
| `composer audit` | `api/` | Avis de sécurité sur les dépendances PHP |
| `npm run test` | `web/` | Tests Vitest |
| `npm run lint` | `web/` | ESLint |
| `npm run typecheck` | `web/` | Vérification des types TypeScript |
| `npx playwright install chromium` | `web/` | Installation du navigateur de test (une fois) |
| `npx playwright test` | `web/` | Tests de bout en bout (démarre l'API et le front si besoin) |

### Réinitialiser la base de développement

Opération destructive (toutes les données de la base `comptoir` sont effacées), à lancer vous-même :

```bash
cd api
php artisan migrate:fresh --seed
```

### Exporter une facture

```bash
cd api
php artisan invoices:export 1
```

Le fichier est écrit dans `api/storage/app/invoices/facture-1.pdf`.

## Intégration continue

Le workflow `.github/workflows/lint.yml` vérifie le formatage de l'API (Pint), ESLint et les types du front à chaque pull request et à chaque push sur `main`.
# comptoir
