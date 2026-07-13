# Technical Specifications

## Software stack

### Frontend (Website)

-   Framework: Next.js (Pages router, static generated site)
    https://nextjs.org/
-   Language: TypeScript
-   Styling: CSS Modules
-   Text to speech: ReadSpeaker (Readspeaker needed)
-   Analytics: Piwik PRO (Piwik PRO needed)

### Backend (CMS)

-   CMS: Laravel Nova 4.x (License needed)
    https://nova.laravel.com/
    Please create your own account and add the license key to the environment variables of this application.
-   Framework: Laravel 11.x
    https://laravel.com/
-   Language: PHP 8.2
-   Database: MySQL 8.0
-   Server requirements: https://laravel.com/docs/11.x/deployment#server-requirements
-   The imagick php extension is required on the server for image processing.
-   API: JSON:API standard, used for fetching content from CMS for generating the frontend
    https://jsonapi.org/
-   Several Laravel Nova packages to extend the functionality of the CMS with features including but not limited to:
    -   Roles and permissions
        https://github.com/serenysoft/nova-permissions
    -   CMS fields for flexible content blocks
        https://github.com/whitecube/nova-flexible-content
    -   Creating pages with parent-child hierarchy with multilanguage support and flexible content blocks
        https://github.com/grrr-amsterdam/nova-pages-tool
    -   Making content fields translatable
        https://github.com/outl1ne/nova-translatable
    -   Attaching images to resources
        https://github.com/ebess/advanced-nova-media-library
    -   Managing site wide settings
        https://github.com/outl1ne/nova-settings
    -   Publishing the static website via GitHub Actions (legacy AWS setup,
        only shown when `STATIC_SITE_PUBLISH_STRATEGY=github`)
        https://github.com/norday-agency/nova-publish
    -   Ability to add notes to resources (used for intakes)
        https://github.com/outl1ne/nova-notes-field
-   PHPStan for static analysis
    https://phpstan.org/

### Other tools

-   Git for version control
-   Github for code hosting
-   Github Actions for CI/CD

## Hosting and architecture

### Background

The application was originally built on Norday's standard AWS stack: the CMS on AWS EC2, the static website stored in an S3 bucket and served via CloudFront, with publication triggered through GitHub Actions. This setup works well when you control the infrastructure.

The foreign government institutions that this application is built for do not permit the use of AWS or any external cloud provider — they require the application to run entirely on their own servers. These servers are typically standard Apache servers managed through a hosting control panel, without SSH access.

This constraint drove two architectural changes:

1. **One-server setup:** instead of a decoupled CMS server + S3/CloudFront, both the CMS and the static website are hosted on the same server. The website source files live in a `website/` subdirectory inside the CMS root. The server exposes three vhosts: one for the CMS (e.g., `admin.imojustmove.org`), one for the static website (e.g., `imojustmove.org`), and one for public media uploads (e.g., `assets.imojustmove.org`). See [DEPLOYMENT.md](./DEPLOYMENT.md) for the vhost structure and document roots.
2. **In-CMS publishing:** instead of GitHub Actions pushing to S3, a Nova Action in the CMS runs `npm run build` on the server and copies the output to the webserver's public directory. This works without SSH or an external pipeline.

This codebase contains the migrated one-server setup for Nigeria Imo State, running on a government server without any AWS dependency.

The application is deployed as separate instances across two distinct hosting scenarios.

### Environments

| Environment | SITE_VARIANT | CMS domain | Frontend domain | Hosting | Architecture |
|---|---|---|---|---|---|
| Norday (interim) | `nigeria-imo-state` | admin.imojustmove.org | imojustmove.org | Norday's server (`hiil-cms1.cust.webslice.eu`) | One-server |
| Imo State government server | `nigeria-imo-state` | admin.imojustmove.org | imojustmove.org | Government server (hosting control panel) | One-server |

In the first phase the application is hosted on Norday's server, as an interim step before handover to the Imo State government server. The domains stay the same; DNS is switched at handover.

### Norday's server (SSH available)

-   **Server:** `hiil-cms1.cust.webslice.eu`, user `hiil`
-   **Deployment:** PHP Deployer via GitHub Actions
-   Hosts the interim Imo State environment until handover to the government server.

### Government server (cPanel only)

After handover, the Imo State production environment is hosted on a server managed by the government. SSH access is not available.

-   **Access:** Hosting control panel (differs per country — no SSH)
-   **Architecture:** One-server — both the CMS and website run on the same server
-   **Deployment:** Manual file upload via cPanel; artisan commands run via cPanel Terminal
-   **Website publishing:** Via the Trigger Static Site Deployment action in the CMS (no GitHub Actions)
-   No AWS services are used on these servers

For detailed deployment procedures for both hosting scenarios, see [DEPLOYMENT.md](./DEPLOYMENT.md).

## Security and performance specifications

### Security

We have several security measures in place to ensure the security of the application.

-   Session-based authentication for CMS users (default Laravel authentication)
-   SPA Authentication to provide user specific data on the static frontend website (Laravel Sanctum)
-   API endpoints of the cms is protected with user specific API tokens (Laravel Sanctum)
-   We use GitHub's Dependabot to keep dependencies up to date. Dependabot is a free GitHub feature that automatically updates your dependencies.

### Authorization

-   CMS users have different roles and permissions to access different parts of the CMS, based on their roles and each role can have a different set of permissions. For all permissions see the policies in the codebase.
-   Data is accessible only to authenticated users with the right permissions. Each content type has it's own policy.
-   Roles can be added and edited in the CMS: administrators can create new roles and assign permissions at will.

#### Permissions

-   MANAGE_INTAKES: allows a user to manage intakes in the CMS
-   MANAGE_OWN_INTAKES: allows someone to manage their own intakes (Assigned to them)
-   MANAGE_REFERRALS: allows someone to manage referrals in the CMS
-   MANAGE_USERS: allows someone to manage users in the CMS
-   MANAGE_SITE_CONTENT: allows someone to create and update pages on the website
-   MANAGE_FEEDBACK: allows someone to view and manage feedback in the CMS
-   MANAGE_ROLES: allows someone to assign roles and assign permissions to roles
-   PUBLISH_SITE: allows someone to publish a new static site
-   ISSUE_API_TOKENS: allows someone to create new API tokens (used to access the API)
-   VIEW_SITE_CONTENT: allows someone to view pages in the CMS (used for the Site Generator role to build the static site)
-   MANAGE_REFERRAL_STATUSES: allows someone to manage referral statuses

### Hosting security

On Norday's server, access is controlled via SSH keys and GitHub Actions secrets. On government servers, access is controlled by the hosting control panel credentials managed by each government.

### Performance and scalability

To serve the site as fast as possible, we use a static generated site with Next.js. The static output (HTML, CSS, JS) can be served from any standard web server. The static files are served directly from the same server as the CMS.

## Analytics integration

We use Piwik PRO for Analytics.
https://piwik.pro/

## Monitoring

We have also integrated Sentry https://sentry.io for the backend (cms) and form handler to monitor errors and exceptions.
Add a `SENTRY_LARAVEL_DSN` to the `.env` file to support your Sentry account.
