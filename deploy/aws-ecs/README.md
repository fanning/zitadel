# Netwire auth (Zitadel) on AWS ECS

`auth.netwire.com` runs this fork's `netwire` branch as one Fargate task in
cluster `prod-hiveskill-com` (us-east-1, account 908590140418), behind the
shared `valuesetters` application load balancer.

| Piece | Value |
|---|---|
| Task definition | `netwire-auth-taskdef.json` (containers `zitadel` :8080 and `login` :3000) |
| Images | ECR `netwire-auth` (from `apps/api/Dockerfile`) and `netwire-auth-login` (from `apps/login/Dockerfile`), tagged `netwire-<git short sha>` |
| Database | Aurora PostgreSQL cluster `valuesetters`, database `netwire_auth_db`, role `auth_service_user`, `sslmode=require` |
| Secrets | Secrets Manager `netwire/zitadel/{masterkey,db-password,admin-password,login-client-pat,login-cookie-secret,system-user-hive-key}` |
| System API user | `hive` (RSA key in `netwire/zitadel/system-user-hive-key`, public key in the task definition as `ZITADEL_SYSTEMAPIUSERS`); signs RS256 JWTs (`iss`=`sub`=`hive`, `aud`=`https://auth.netwire.com`) for admin/system API automation such as branding |
| ALB rules (443) | prio 30 `auth.netwire.com` + `/ui/v2/login*` -> tg `netwire-auth-login`; prio 31 `/` -> 302 `/ui/console`; prio 32 host -> tg `netwire-auth`. The legacy `netwire.com, *.netwire.com` rule was moved from prio 18 to 33 so it no longer shadows this host. |
| DNS | Netlify zone `netwire.com`, CNAME `auth` -> ALB hostname |
| Logs | CloudWatch `/ecs/netwire-auth`, `/ecs/netwire-auth-login` |

## Build and push

```bash
. ~/.nvm/nvm.sh && nvm use 24
pnpm install --frozen-lockfile
pnpm nx run @zitadel/api:pack        # -> zitadel/zitadel:local
pnpm nx run @zitadel/login:pack      # -> zitadel/zitadel-login:local
TAG=netwire-$(git rev-parse --short HEAD)
ECR=908590140418.dkr.ecr.us-east-1.amazonaws.com
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR
docker tag zitadel/zitadel:local       $ECR/netwire-auth:$TAG       && docker push $ECR/netwire-auth:$TAG
docker tag zitadel/zitadel-login:local $ECR/netwire-auth-login:$TAG && docker push $ECR/netwire-auth-login:$TAG
```

## First-time database setup

The first instance (org `Netwire`, human `admin`, machine `login-client`) is
created once by running `zitadel init` and `zitadel setup` from an operator
host that can reach the cluster, with the same environment as the task
definition. `setup` writes the login client's personal access token to the
path in `ZITADEL_FIRSTINSTANCE_LOGINCLIENTPATPATH`; that value is stored as
Secrets Manager `netwire/zitadel/login-client-pat` and injected into the
`login` container as `ZITADEL_SERVICE_USER_TOKEN`. The running task uses
`start-from-setup`, so later releases apply their own migrations on start.

## Release

```bash
sed -i "s#\(netwire-auth\(-login\)\?\):netwire-[0-9a-f]*#\1:$TAG#" deploy/aws-ecs/netwire-auth-taskdef.json
aws ecs register-task-definition --cli-input-json file://deploy/aws-ecs/netwire-auth-taskdef.json
aws ecs update-service --cluster prod-hiveskill-com --service netwire-auth --task-definition netwire-auth
aws ecs wait services-stable --cluster prod-hiveskill-com --services netwire-auth
curl -s https://auth.netwire.com/.well-known/openid-configuration | jq .issuer
```

Keep this branch rebased on upstream release tags (`git fetch upstream --tags`);
the rebrand commit touches only assets, manifests, titles and locale strings.

## Known upstream quirk

v4.20.0's login build copies `public/` to the standalone root while the custom
server serves static files from the app directory, so favicons and logos 404 in
the stock image. The login Dockerfile in this fork copies `public` beside the
server as well.
