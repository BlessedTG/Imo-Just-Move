# Development environment

## Repository structure

```
hiil/
├── cms/        # Laravel Nova CMS (PHP 8.2, Laravel 11)
└── website/    # Next.js website (TypeScript, statically generated)
```

---

## CMS local setup

### Requirements

- PHP 8.2
- Composer
- Node.js 20.x + npm (for Nova assets)
- MySQL 8.0
- The `imagick` PHP extension

### Steps

1. Navigate to the CMS directory:
   ```bash
   cd cms
   ```

2. Install PHP dependencies:
   ```bash
   composer install
   ```

3. Install Node dependencies (required for Nova asset compilation):
   ```bash
   npm install
   ```

4. Copy the example environment file and configure it:
   ```bash
   cp .env.example .env
   ```
   Set the following values in `.env`:
   - `DB_*` — your local database credentials
   - `NOVA_LICENSE_KEY` — your Nova license key (see note below)
   - `SITE_VARIANT` — one of `tunisia`, `nigeria-ogun-state`, `nigeria-imo-state`
   - `API_TOKEN` — a token of your choice; the website uses this to authenticate API requests

5. Generate the application key:
   ```bash
   php artisan key:generate
   ```

6. Create the storage symlink:
   ```bash
   php artisan storage:link
   ```

7. Run migrations and seeders:
   ```bash
   php artisan migrate
   php artisan app:seed-settings
   php artisan db:seed --class=RolesAndPermissionsSeeder
   php artisan db:seed --class=IntakeStatusSeeder
   php artisan db:seed --class=ReferralStatusSeeder
   ```

8. Start the development server:
   ```bash
   php artisan serve
   ```
   The CMS will be available at `http://localhost:8000`.

   If you need to work on Nova assets, also run Vite in a separate terminal:
   ```bash
   npm run dev
   ```

### Nova license key

Laravel Nova requires a paid license. Set your license key as `NOVA_LICENSE_KEY` in the CMS `.env` file. You can find or generate license keys at [nova.laravel.com](https://nova.laravel.com). If you do not have access to the license, contact the project owner.

### Running tests and static analysis

```bash
composer test                      # PHPUnit
./vendor/bin/phpstan analyse       # PHPStan (uses phpstan-baseline.neon)
prettier --write .                 # Prettier (includes PHP via @prettier/plugin-php)
```

---

## Website local setup

### Requirements

- Node.js 20.x
- npm
- A running local CMS instance (see above)

### Steps

1. Navigate to the website directory:
   ```bash
   cd website
   ```

2. Install Node dependencies:
   ```bash
   npm install
   ```

3. Copy the example environment file and configure it:
   ```bash
   cp .env.example .env
   ```
   Set the following values in `.env`:
   - `NEXT_PUBLIC_API_ROOT` — URL of your local CMS API, e.g. `http://localhost:8000/api`
   - `API_TOKEN` — must match the `API_TOKEN` set in the CMS `.env`
   - `SITE_VARIANT` — one of `tunisia`, `nigeria-ogun-state`, `nigeria-imo-state`
   - `NEXT_PUBLIC_FRONTEND_URL` — e.g. `http://localhost:3000`
   - `NEXT_PUBLIC_ALLOW_CRAWLERS` — set to `false` for local development

4. Start the development server:
   ```bash
   npm run dev
   ```
   The website will be available at `http://localhost:3000`.

### Running tests and linting

```bash
npm test          # Jest unit tests
npm run lint      # ESLint + Stylelint + TypeScript check
npm run format    # Prettier
```

---

## Deployment

For instructions on deploying to production or staging environments, see [DEPLOYMENT.md](./DEPLOYMENT.md).
