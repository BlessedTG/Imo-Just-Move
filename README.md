# HiiL Justice Advise Tool

The HiiL Justice Advise Tool is a web application that helps citizens find legal information and advice. It is built for HiiL (Human Development Justice Institute) and deployed for several foreign government institutions.

## Site variants

The application is deployed as separate instances, each for a different institution:

| Variant            | SITE_VARIANT         | Frontend domain      | CMS domain                 |
| ------------------ | -------------------- | -------------------- | -------------------------- |
| Tunisia            | `tunisia`            | moustacharchogli.org | admin.moustacharchogli.org |
| Nigeria Ogun State | `nigeria-ogun-state` | ibiisadi.org         | admin.ibiisadi.org         |
| Nigeria Imo State  | `nigeria-imo-state`  | imojustmove.org      | admin.imojustmove.org      |

## Architecture

The project is a monorepo with two components:

- **`cms/`** — Laravel Nova v4 (PHP 8.2, Laravel 11), MySQL 8.0. Manages content, intakes, referrals, and feedback. Exposes a read-only JSON:API for the website.
- **`website/`** — Next.js (Pages router, statically generated) in TypeScript. Fetches content from the CMS API at build time.

### Hosting

There are two hosting scenarios:

**Government servers (control panel only, manual deployment)**
The production environments (Tunisia, Nigeria Ogun State, Nigeria Imo State) are hosted on servers managed by the respective governments. No SSH access is available. Both CMS and website run on the same server, in a one-server setup with separate vhosts for the CMS, the website, and media uploads. Deployments are done manually via a control panel.

**Norday's server (SSH access, GitHub Actions deployment)**
Hosts the staging environments.

The one-server setup and in-CMS publish mechanism exist because the government servers do not permit AWS or external cloud services. See [TECHNICAL_SPECIFICATIONS.md](./TECHNICAL_SPECIFICATIONS.md#background) for the full background.

## Getting started

- **Local development:** see [DEVELOPMENT_ENVIRONMENT.md](./DEVELOPMENT_ENVIRONMENT.md)
- **Deployment:** see [DEPLOYMENT.md](./DEPLOYMENT.md)
- **Technical specifications:** see [TECHNICAL_SPECIFICATIONS.md](./TECHNICAL_SPECIFICATIONS.md)

## Component documentation

- **CMS:** see [cms/README.md](./cms/README.md)
- **Website:** see [website/README.md](./website/README.md)
