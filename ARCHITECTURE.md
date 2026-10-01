# ARCHITECTURE — k8s-shopify-framework-pocharlies

Base kustomize compartida para desplegar apps Shopify embebidas de Skirmshop en k3s vía ArgoCD. Es el lado de despliegue de `pocharlies-org/shopify-app-framework`.

## Clientes y versiones
- Sin cliente propio. No es una Application de ArgoCD: los consumidores la referencian como base remota. Única rama y tronco real: `deploy/prod` (org `pocharlies`, no `pocharlies-org`).

## Dependencias (ambos sentidos)
- Lo consumen con `resources: github.com/pocharlies/k8s-shopify-framework-pocharlies//base?ref=deploy/prod`: `k8s-shopify-affiliate`, `-bundles`, `-collections-tree`, `-picker`, `-sii`, `-translations` y `pocharlies/k8s-shopify-serial-numbers`. Un push a `deploy/prod` cambia las 7 en su siguiente render.
- Depende del cluster: namespace `skirmshop`, Postgres compartido `postgres-shared-rw.databases.svc.cluster.local`, secret `shared-postgres-app`, Traefik (`IngressRoute`, TLS), `external-dns`, Keycloak (`components/forward-auth`, middleware `sso-chain` en ns `keycloak`).
- Gemelo de código: `pocharlies-org/shopify-app-framework`.

## Stack
Kustomize (`base/`, `components/`, `examples/`). Sin Helm. Sin CI propio.

## Componentes compartidos
- `base/`: Deployment `app` (label `app.kubernetes.io/name: shopify-app`, imagen placeholder `shopify-app:latest` sustituida por `images:` en cada overlay, puerto 3000, `readinessProbe` tcp), Service e IngressRoute; `base/kustomizeconfig/traefik-namereference.yaml` hace que `namePrefix` reescriba las referencias de Traefik.
- `components/forward-auth/`: protege un backend de admin con SSO (NO usar en apps embebidas: rompe OAuth/iframe).
- `examples/`: overlays de ejemplo (bundles, sii, collections-tree, affiliate, translations, picker-admin) con base relativa.

## Cómo se construye
Una app nueva = `k8s/kustomization.yaml` en su repo con `namePrefix`, `labels` (instancia única para el selector), `images:` por digest, parche de entorno (`SHOPIFY_APP_URL`, `SCOPES`, `DATABASE_URL`) y, si no es 3000, parche de puerto y de `match` del IngressRoute (`Host(skirmshop.e-dani.com) && PathPrefix(/<app>)`). README.md del repo lo detalla.

## Tests y validaciones
Ninguno en este repo. Se valida de rebote en el CI de cada consumidor (`reusable-ci.yml` renderiza kustomize con kubeconform), pero no antes de empujar aquí.

## CI/CD y despliegue
No tiene `.github/`. Se aplica al sincronizar la Application de cada consumidor.

## Decisiones y trampas
- `ref=deploy/prod` móvil: sin tags, cualquier cambio de la base llega a producción de 7 apps; propuesta: etiquetar y fijar.
- `examples/` duplica overlays vivos y puede divergir de ellos.
- Repo en la org `pocharlies`: el CI de los consumidores configura auth git para «private kustomize bases».
- La conexión Prisma por defecto dimensiona el pool con las CPU del host; los overlays fijan `connection_limit=5`.
