# Level 2 Setup For `retail`

This is the exact workflow for this repo.

## Repo Model

Use two repos on the server:

```text
/home/jimmy/work/
├── frappe_docker/
└── retail/
```

- `frappe_docker` is the development environment
- `retail` is the real app repo

## Important Rule

The app directory inside the dev container must point to the real `retail` repo.

Good:

- `/home/jimmy/work/retail`
- bind-mounted to `/workspace/development/frappe-bench/apps/retail`

Bad:

- editing a second copy of `retail` under `frappe_docker/development/frappe-bench/apps/retail`

## How Live Editing Works

Your dev container now needs this extra bind mount in `frappe_docker/.devcontainer/docker-compose.yml`:

```yaml
services:
  frappe:
    volumes:
      - ..:/workspace:cached
      - /home/jimmy/work/retail:/workspace/development/frappe-bench/apps/retail
```

After changing that file, rebuild or recreate the dev container.

When this bind mount exists:

- editing `/home/jimmy/work/retail` on the host changes the live app code inside the container
- Frappe sees the edits immediately because the container path is the same app path bench is using

## First-Time Dev Setup

### 1. Clone the repos

```bash
mkdir -p ~/work
cd ~/work
git clone https://github.com/frappe/frappe_docker.git
git clone https://github.com/00JIMMY00/retail.git
```

### 2. Prepare `frappe_docker`

```bash
cd ~/work/frappe_docker
cp -R devcontainer-example .devcontainer
cp -R development/vscode-example development/.vscode
```

### 3. Add the retail bind mount

Edit `.devcontainer/docker-compose.yml` and add:

```yaml
- /home/jimmy/work/retail:/workspace/development/frappe-bench/apps/retail
```

inside the `frappe` service `volumes:` block.

### 4. Open in VS Code

```bash
cd ~/work/frappe_docker
code .
```

Then run:

- `Dev Containers: Reopen in Container`

### 5. Build the bench

Inside the dev container:

```bash
cd /workspace/development
python installer.py -v
```

### 6. Install the app on the site

Inside the dev container:

```bash
cd /workspace/development/frappe-bench
bench --site development.localhost install-app retail
```

### 7. Start development

```bash
cd /workspace/development/frappe-bench
bench start
```

Now edit files in:

```text
/home/jimmy/work/retail
```

or from VS Code at:

```text
development/frappe-bench/apps/retail
```

They are the same live app path once the bind mount is active.

## Push Changes To GitHub

Inside the app repo:

```bash
cd /home/jimmy/work/retail
git status
git add .
git commit -m "feat: describe your change"
git push origin test
```

## GitHub Actions Deploy Setup

This repo includes `.github/workflows/deploy.yml`.

It does two things:

- auto-deploys when you push to `test`
- supports manual deploy from `Actions > Deploy Retail > Run workflow`

## Required GitHub Environment

Create a GitHub environment named:

- `production`

Add these environment secrets:

- `PROD_HOST`
- `PROD_PORT`
- `PROD_USER`
- `PROD_SSH_KEY`
- `BENCH_PATH`
- `SITE_NAME`

Example values:

- `PROD_HOST`: your server IP or hostname
- `PROD_PORT`: `22`
- `PROD_USER`: the Linux user that owns the bench
- `PROD_SSH_KEY`: private key used by GitHub Actions to SSH to the server
- `BENCH_PATH`: `/home/jimmy/work/frappe_docker/development/frappe-bench`
- `SITE_NAME`: `development.localhost` for a dev server, or your real production site name

## What The Deploy Workflow Does

On the server it runs:

1. `git fetch` in `apps/retail`
2. `git checkout <branch>`
3. `git pull --ff-only`
4. `bench --site <site> backup --with-files`
5. `bench --site <site> migrate`
6. `bench build --app retail`
7. `bench clear-cache`
8. `bench restart`

## Recommended Promotion Model

For safety, use this branch model:

- `test`: remote dev or staging auto-deploy
- `main`: production release branch

Once you are happy, you can duplicate the workflow and make a production variant that deploys only from `main` with manual approval.

## Notes

- `bench restart` may require extra server permissions depending on your production setup.
- If restart needs `sudo`, configure that on the server side rather than storing a sudo password in GitHub.
- Keep production secrets only in GitHub environment secrets, never in repo files.
