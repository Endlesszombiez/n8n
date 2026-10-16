# Build and run this fork with Docker Compose

This repository already has Dockerfiles and application build scripts.
The upstream build workflow uses n8n infrastructure and registry settings.
The upstream Compose examples pull official n8n images.
Those images do not include changes in this fork.

Use the fork workflow to build this source and publish two images to GitHub
Container Registry (GHCR). Then use Compose to pull and run them.
The workflow publishes Linux AMD64 images for x86-64 hosts.
It does not publish ARM64 images for Raspberry Pi or ARM servers.

## Publish the images

1. Commit and push the fork deployment files to your fork's default branch.
2. Open **Actions** in your GitHub fork. Enable workflows if GitHub asks.
3. Select **Docker: Publish fork**. Select **Run workflow**.
4. Select the branch that contains your changes. Start the workflow.
5. Wait for the workflow to finish. Read the image names in its summary.

The workflow uses GitHub-hosted runners and the automatic `GITHUB_TOKEN`.
It needs permission to write packages. It does not need n8n registry secrets
or a Blacksmith account. GitHub Actions usage limits still apply.
The build can take a long time because it compiles the full application.

For this fork, the image names are:

- `ghcr.io/endlesszombiez/n8n:fork`
- `ghcr.io/endlesszombiez/n8n-runners:fork`

Each run also publishes `sha-<full-commit-sha>` tags for both images.
The `fork` tag points to the last successful manual build, from any selected branch.
No image is published when you only push a commit.
Use a commit tag to select a fixed build.
Both services must use the same tag.

New GHCR packages can be private. To allow anonymous pulls, open each package
on GitHub. Open **Package settings** and change its visibility to **Public**.
For private packages, run `docker login ghcr.io` on the deployment host.
Enter your GitHub username and a personal access token with `read:packages`
at the prompts. Do not put the token in a command or in the Compose file.
If publishing fails with a permission error, check the repository's Actions
policy and the package's Actions access settings.

## Start with Compose

Install Docker Engine or Docker Desktop with Docker Compose v2.
Use Linux containers. You do not need Node.js or a source checkout on this host.

1. Create a deployment directory.
2. Copy [compose.fork.yml](compose.fork.yml) into it as `compose.yaml`.
3. Copy [fork.env.example](fork.env.example) into it as `.env`.
4. Set `N8N_RUNNERS_AUTH_TOKEN` in `.env` to a random secret of at least 32 characters.
5. Set the image names if your GitHub owner or repository name differs.
   GHCR image names must use lowercase letters.

Keep `.env` private. Keep it outside source control.
You can generate the runner secret with a password manager or `openssl rand -hex 32`.

Run these commands from the deployment directory:

```sh
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose logs -f n8n runners
```

Open <http://localhost:5678>. Create the owner account on the first start.
The port binds to the deployment host's loopback interface.
For a remote host, use an SSH tunnel, such as
`ssh -L 5678:localhost:5678 user@server`, then open the same local URL.
For public access, configure an HTTPS reverse proxy and the n8n public URL
settings. See the [hosting guide](https://docs.n8n.io/hosting/).

The named volume stores the SQLite database and n8n settings, including the
credential encryption key. Keep the volume when you replace the containers.
The runner connects through the Compose network. Its broker port is not
published on the host.
This setup provides n8n and its Code node runners. The optional sandbox
services from `get-n8n-compose.yml` are not included.

## Update or select a fixed build

Publish a new build with **Docker: Publish fork**.
For a fixed build, set `N8N_IMAGE_TAG=sha-<full-commit-sha>` in `.env`.
For the most recent manual build, use `N8N_IMAGE_TAG=fork`.
Back up the data volume before an update. Then run:

```sh
docker compose pull
docker compose up -d
docker compose logs --tail=100 n8n runners
```

Use `docker compose down` to stop and remove the containers.
Do not add `--volumes` or `-v` unless you intend to delete the stored data.
A previous image can be selected with its commit tag. Database migrations can
prevent a downgrade. Restore the matching backup if a downgrade requires it.
Keep the same deployment directory and Compose project name to reuse the volume.
When you move an existing instance, migrate its data and encryption keys first.

## Build locally instead

Use Linux or WSL2 with Docker integration. The build scripts use POSIX tools.
Install Node.js 26.10.0 and the pnpm version from the root `packageManager`
field (currently 12.4.2). Install Python 3, make, and a C/C++ compiler for native
dependencies. The Docker build also needs the Compose and Buildx plugins.
Allow enough memory and disk space for the full monorepo build.

From the repository root, run:

```sh
pnpm install --frozen-lockfile
pnpm build:docker > build.log 2>&1
```

If the build fails, inspect `build.log`.
This command compiles the application into `compiled/` and the JavaScript
runner into `dist/task-runner-javascript/`. It then builds `n8nio/n8n:local`
and `n8nio/runners:local` for the host architecture.
The main Dockerfile requires these compiled files. It cannot build a fresh
source checkout by itself.

Use these values in the deployment `.env`:

```dotenv
N8N_IMAGE=n8nio/n8n
N8N_RUNNERS_IMAGE=n8nio/runners
N8N_IMAGE_TAG=local
```

Run `docker compose up -d --pull never` on the same Docker host.
Skip `docker compose pull` for these local images.
To update, rebuild the images and run `docker compose up -d --pull never` again.
