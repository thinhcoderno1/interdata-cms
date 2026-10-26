# Interdata CMS

Payload CMS with Next.js and PostgreSQL.

## Local development

1. Install Node.js matching `package.json` and pnpm `10.34.9`.
2. Copy `.env.example` to `.env` and set a PostgreSQL `DATABASE_URL` and a long,
   random `PAYLOAD_SECRET`.
3. Run `pnpm install --frozen-lockfile`, then `pnpm dev`.
4. Open <http://localhost:3000/admin>.

Keep production credentials separate from local and test credentials.

## Production deployment with Docker Compose

The repository's `docker-compose.yml` builds the application from the checked-out
Git repository using `Dockerfile`. It runs the Next.js standalone production
server on `0.0.0.0:3000`; it does not run `pnpm dev` or mount the source directory
from the host.

In Coolify or another Compose hosting platform:

1. Select this Git repository, branch `main`, and `docker-compose.yml`.
   The build context must be the repository root containing `package.json`.
   If using a pasted Compose definition, the platform must still check out the
   repository so `build.context: .` has the source files.
2. Add `DATABASE_URL` and `PAYLOAD_SECRET` in the deployment's environment settings.
   Compose requires both values and explicitly passes them into the container.
   `DATABASE_URL` must reach an existing PostgreSQL service from the app container;
   `localhost` refers to the app container itself. Configure the platform's Docker
   network so the app can reach your PostgreSQL service.
3. Route the domain to service `payload`, internal port `3000`, using HTTP to
   the app container. The reverse proxy handles external HTTPS.
4. Remove any platform command override containing `pnpm dev` and any source or
   `node_modules` mount from the previous configuration. Rebuild and redeploy
   from Git; restarting the old container does not apply these changes.

For a local Docker installation, `docker compose up --build -d` starts the app.
The Compose file exposes port `3000` only inside Docker. To access it directly
from the host without a reverse proxy, add a local port mapping such as
`127.0.0.1:3000:3000` to service `payload`.

The `media` named volume persists uploaded files. PostgreSQL is managed separately;
this Compose file does not create or migrate a database. For a new production
schema, generate and review Payload migrations and run them against the intended
database as part of your release process.

## Environment variables and secrets

`src/payload.config.ts` reads `DATABASE_URL` and `PAYLOAD_SECRET` from the server
environment. Keep the production secret stable across redeployments.

- `.env.example` contains placeholders only.
- `.gitignore` excludes `.env` and `.env.*`, except `.env.example`.
- `.dockerignore` excludes real environment files from the Docker build context.
- Never prefix either secret with `NEXT_PUBLIC_` or put real credentials in Git,
  Dockerfile `ARG` / `ENV` instructions, or the Compose file.
- The current app is built without production credentials; the deployment injects
  them at runtime. If future static generation needs secrets during a build, use
  BuildKit secret mounts explicitly rather than Docker build arguments.
- Ignore rules do not remove secrets from existing Git history.

## Troubleshooting

`Command "dev" not found` with an almost-empty `pnpm install` usually means the
container is running in a directory without the project's `package.json` scripts.
Check the deployment source and build context. The production configuration in
this repository avoids the old `.:/home/node/app` source mount and pins pnpm.

A 502 response does not by itself prove a port mismatch. Check build logs first,
then runtime logs, and confirm that the production server is listening on port
`3000`. Database connectivity and schema errors also need to be resolved before
Payload can serve requests successfully.

References: [Payload deployment](https://payloadcms.com/docs/production/deployment),
[Payload migrations](https://payloadcms.com/docs/database/migrations),
[Coolify environment variables](https://coolify.io/docs/applications/configuration/environment-variables),
[Docker bind mounts](https://docs.docker.com/engine/storage/bind-mounts/).
