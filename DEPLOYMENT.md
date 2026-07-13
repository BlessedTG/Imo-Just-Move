# Deployment

## Environments

| Environment | SITE_VARIANT | CMS domain | Frontend domain | Hosting | Deployment method |
|---|---|---|---|---|---|
| Norday (interim) | `nigeria-imo-state` | admin.imojustmove.org | imojustmove.org | Norday's server (`hiil-cms1.cust.webslice.eu`) | PHP Deployer via GitHub Actions |
| Imo State government server | `nigeria-imo-state` | admin.imojustmove.org | imojustmove.org | Government server | Manual via hosting control panel |

In the first phase the application is hosted on Norday's server, as an interim step before handover to the Imo State government server. The domains stay the same; DNS is switched at handover.

---

## Deploying to Norday's server (SSH available)

**Server:** `hiil-cms1.cust.webslice.eu`, user `hiil`

Norday's server hosts the interim Imo State environment, until the application is handed over to the Imo State government server. Deployments are handled by [PHP Deployer](https://deployer.org/) triggered via GitHub Actions.

### Triggering a deploy

Each environment has a dedicated workflow file (e.g., `deploy-cms-imojustmove-production.yml`). Trigger a deployment by running the workflow manually (`workflow_dispatch`) from the GitHub Actions tab, typically on a release tag.

### What Deployer does

Deployer runs the following steps (see `cms/deploy.php` for full details):

1. Prepares a new release directory on the server
2. Authenticates with nova.laravel.com using `NOVA_LICENSE_KEY`
3. Runs `composer install --no-dev --optimize-autoloader`
4. Runs `php artisan storage:link`
5. Writes the current `APP_VERSION` to `.env`
6. Runs `php artisan optimize`
7. Runs database migrations (`php artisan migrate --force`)
8. Runs seeders: `app:seed-settings`, `RolesAndPermissionsSeeder`, `IntakeStatusSeeder`, `ReferralStatusSeeder`
9. Creates the symlink for the new release (atomic deploy)
10. Reloads PHP-FPM (`sudo service php82-php-fpm reload`)
11. Restarts the queue worker (`php artisan queue:restart`)

For hosts labeled with the `one_server_with_multiple_vhosts` hosting strategy in `cms/deploy.php`, website files are also uploaded and `npm install` is run on the server during deployment.

---

## Deploying to a government server (no SSH)

After handover, the Imo State production environment is hosted on a server managed by the government. No SSH access is available — only access through the hosting control panel. All deployment steps must be performed manually using the control panel's file manager and terminal.

### Vhost structure

The one-server setup requires three vhosts on the same server. Using Nigeria Imo State as an example:

| Vhost | Example domain | Document root |
|---|---|---|
| CMS | `admin.imojustmove.org` | `{cms-root}/public` |
| Website | `imojustmove.org` | The path configured as `STATIC_SITE_PUBLIC_PATH` |
| Media | `assets.imojustmove.org` | The path configured as `PUBLIC_MEDIA_DISK_ROOT` |

- The **CMS vhost** serves the Laravel application (Nova admin and the JSON:API).
- The **website vhost** serves the static site. The Nova publish action copies the Next.js build output (`/out`) to `STATIC_SITE_PUBLIC_PATH`, so that path must be the document root of this vhost.
- The **media vhost** serves the publicly uploaded media files (images, attachments). `PUBLIC_MEDIA_DISK_URL` must be set to the public URL of this vhost (e.g., `https://assets.imojustmove.org`) so the CMS generates correct media URLs.

### Initial setup on a new government server

Follow these steps when setting up the application on a government server for the first time:

1. Upload the contents of the `cms/` directory from the repository to the server. This becomes the CMS root directory. Then upload the contents of the `website/` directory from the repository into a `website/` subdirectory inside that CMS root. The resulting structure on the server should be:
   ```
   {cms-root}/
   ├── app/
   ├── ...
   └── website/
       ├── components/
       ├── pages/
       └── ...
   ```
2. Create a `.env` file in the CMS root based on `cms/.env.example`. Set at minimum: `APP_KEY`, `APP_ENV`, `APP_URL`, `DB_*`, `NOVA_LICENSE_KEY`, `SITE_VARIANT`, `FRONTEND_URL`, `QUEUE_WORKER_VIA_CRON=true`, `STATIC_SITE_PUBLISH_STRATEGY=local`, and `STATIC_SITE_PUBLIC_PATH` (the absolute server path where the built website should be served from). Leave `API_TOKEN` blank for now — you will generate it in step 11.
3. Create a `.env` file for the website at `website/.env` inside the CMS root, based on `website/.env.example`. Set at minimum: `NEXT_PUBLIC_API_ROOT`, `SITE_VARIANT`, `NEXT_PUBLIC_FRONTEND_URL`, and `NEXT_PUBLIC_ALLOW_CRAWLERS`. Leave `API_TOKEN` blank for now — you will fill it in after step 11. See [Environment variables](#environment-variables) for descriptions.
4. In the hosting control panel terminal, navigate to the CMS root and authenticate with Nova:
   ```
   composer run post-autoload-dump
   ```
   Ensure `NOVA_LICENSE_KEY` is set correctly in `.env` before this step.
5. Install PHP dependencies:
   ```
   composer install --no-dev --optimize-autoloader
   ```
6. Generate the application key (only on first setup — do not regenerate on updates):
   ```
   php artisan key:generate
   ```
7. Create the storage symlink:
   ```
   php artisan storage:link
   ```
8. Run migrations and seeders:
   ```
   php artisan migrate --force
   php artisan app:seed-settings
   php artisan db:seed --class=RolesAndPermissionsSeeder
   php artisan db:seed --class=IntakeStatusSeeder
   php artisan db:seed --class=ReferralStatusSeeder
   ```
9. Optimize the application:
   ```
   php artisan optimize
   ```
10. Set up the cron job to run the Laravel scheduler every minute:
    ```
    * * * * * php /path/to/cms/artisan schedule:run >> /dev/null 2>&1
    ```
11. Create the first admin user and generate the API token for the website:
    - Create an admin user via the terminal:
      ```
      php artisan nova:user
      php artisan assign:role   # select Superadmin
      ```
    - Log in to the CMS at `APP_URL/admin`.
    - Create a second user dedicated to API access (e.g. `site-generator@example.com`). Assign it a role that includes the `VIEW_SITE_CONTENT` permission.
    - Generate an API token for this user: go to the user's detail page in Nova and use the **API Tokens** section.
    - Copy the generated token and set it as `API_TOKEN` in both the CMS `.env` and `website/.env`.
12. Install Node dependencies for the website:
    ```
    cd {cms-root}/website && npm install
    ```
13. Trigger the first website publish using the Nova Publish action (see [Publishing the static website](#publishing-the-static-website) below).

### Deploying an update or critical fix

Follow these steps when deploying a code update to an existing government server installation:

1. Upload the changed files to the server via the hosting control panel file manager, replacing the existing files. This applies to both `cms/` and `website/` depending on what changed.
2. If `cms/composer.json` or `cms/composer.lock` changed, reinstall PHP dependencies:
   ```
   composer install --no-dev --optimize-autoloader
   ```
3. If the update includes new database migrations, run:
   ```
   php artisan migrate --force
   ```
4. Clear and rebuild the application cache:
   ```
   php artisan optimize
   ```
5. Restart the queue worker to pick up any code changes:
   ```
   php artisan queue:restart
   ```
6. If `website/package.json` or `website/package-lock.json` changed, reinstall Node dependencies:
   ```
   cd {cms-root}/website && npm install
   ```
7. If any files in `website/` changed, trigger a new website publish from the CMS Nova Action (see below).

---

## Queue processing

The application uses the `database` queue driver. Jobs are stored in the `jobs` table and must be processed by a worker.

On servers with SSH access (Norday's server), a persistent queue worker can be managed by Supervisor. On servers without SSH access (government servers), there is no persistent worker. Instead, the Laravel scheduler runs `queue:work --stop-when-empty` every minute via the cron job.

This mode is controlled by the `QUEUE_WORKER_VIA_CRON` variable in the CMS `.env`:

- `QUEUE_WORKER_VIA_CRON=true` — the scheduler processes the queue every minute. **Required on all one-server setups (government servers).**
- `QUEUE_WORKER_VIA_CRON=false` (default) — assumes an external persistent worker (e.g. Supervisor) is handling the queue.

When `QUEUE_WORKER_VIA_CRON=true`, jobs are not processed immediately — they run on the next scheduler tick (within one minute). This affects the static site publish job and media scaling jobs, both of which are dispatched to the queue.

The scheduler also runs these tasks regardless of the queue mode:

| Task | Frequency |
|---|---|
| `app:import-intakes` | Every 5 minutes |
| `app:import-feedback` | Every 5 minutes |
| `app:import-referral-requests` | Every 5 minutes |
| `debugbar:clear` | Daily at 01:00 |

All of these require the cron job from the initial setup to be running:
```
* * * * * php /path/to/cms/artisan schedule:run >> /dev/null 2>&1
```

---

## Publishing the static website

The static website is published from within the CMS using the **Trigger Static Site Deployment** action on the **Static Site Deployments** resource in Nova. This requires `STATIC_SITE_PUBLISH_STRATEGY=local`.

When triggered, the action dispatches a queued job that:

1. Runs `npm run build` in the `website/` directory on the server
2. Copies the generated `/out` directory to `STATIC_SITE_PUBLIC_PATH`, replacing the previous version

Each publish is recorded as a deployment with a status (pending, running, success, failed), a message, and its duration. Only one deployment can be active at a time. Because the job runs on the queue, the cron-based queue worker must be running (`QUEUE_WORKER_VIA_CRON=true`).

### How to trigger it

1. Log in to the CMS at the relevant admin domain.
2. Navigate to **Static Site Deployments** in the Nova sidebar.
3. Run the **Trigger Static Site Deployment** action and confirm.

A CMS user must have the `PUBLISH_SITE` permission to trigger this action.

### When to use it

Trigger a publish whenever:
- Frontend content or code has changed and the site needs to be rebuilt
- After an initial server setup to generate the first version of the site

---

## Environment variables

### CMS (`.env`)

| Variable | Description |
|---|---|
| `APP_KEY` | Laravel application encryption key. Generate with `php artisan key:generate`. |
| `APP_ENV` | Application environment. Use `production` on live servers. |
| `APP_URL` | Full URL of the CMS (e.g., `https://admin.imojustmove.org`). |
| `DB_HOST` | Database host. |
| `DB_PORT` | Database port (default: `3306`). |
| `DB_DATABASE` | Database name. |
| `DB_USERNAME` | Database username. |
| `DB_PASSWORD` | Database password. |
| `NOVA_LICENSE_KEY` | Laravel Nova license key. Required for Nova to function. |
| `SITE_VARIANT` | Identifies which site variant this instance serves. Use `nigeria-imo-state` for Imo State. |
| `FRONTEND_URL` | Public URL of the frontend website. Used internally by the CMS (e.g. for display links in Nova). |
| `STATIC_SITE_PUBLISH_STRATEGY` | How the static site is published. Use `local` for one-server setups (government servers). Use `github` for deployments that publish via GitHub Actions to S3 + CloudFront (legacy AWS setup). Default: `github`. |
| `STATIC_SITE_PUBLIC_PATH` | Absolute path on the server where the built static site (`/out`) should be copied. Required when `STATIC_SITE_PUBLISH_STRATEGY=local`. |
| `PUBLIC_MEDIA_DISK_ROOT` | Absolute path on the server where publicly uploaded media files are stored. Should be the document root of the media vhost. |
| `PUBLIC_MEDIA_DISK_URL` | Public URL of the media vhost (e.g., `https://assets.imojustmove.org`). Used by the CMS to generate media URLs. |
| `QUEUE_WORKER_VIA_CRON` | Set to `true` to process the queue via the Laravel scheduler instead of a persistent worker. Required on all servers without Supervisor (government servers). Default: `false`. |
| `SENTRY_LARAVEL_DSN` | Sentry DSN for backend error monitoring. Optional. |
| `API_TOKEN` | Token used by the website to authenticate read-only API requests to the CMS. |

### Website (`.env`)

| Variable | Description |
|---|---|
| `NEXT_PUBLIC_API_ROOT` | Full URL of the CMS JSON:API endpoint (e.g., `https://admin.imojustmove.org/api`). |
| `API_TOKEN` | Token used to authenticate requests to the CMS API. Must match the `API_TOKEN` set in the CMS. |
| `SITE_VARIANT` | Identifies which site variant this build targets. Used for theming and conditional logic. |
| `NEXT_PUBLIC_FRONTEND_URL` | Public URL of the frontend website. Used for sitemap generation and canonical URLs. |
| `NEXT_PUBLIC_ALLOW_CRAWLERS` | Set to `true` to allow search engine indexing. Set to `false` on staging environments. |
