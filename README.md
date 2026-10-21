La CI commune marche avec n'importe quel projet qui respecte trois règles. Solarsys les respecte déjà toutes.
1. Les scripts du package.json disent comment. La CI lance lint, format:check, typecheck (ou type-check), test, build et, en option, test:e2e. Si un script n'existe pas, l'étape est sautée sans erreur (pnpm run --if-present).
2. Un Dockerfile à la racine écrit dist/version.json. C'est ce fichier qui prouve que la prod sert le bon build. Il suffit de deux lignes après le build (comme dans solarsys) :
ARG APP_VERSION=dev
RUN pnpm build && printf '{"version":"%s"}\n' "$APP_VERSION" > dist/version.json
3. Dokploy tire l'image, il ne la construit pas. L'app Dokploy est en provider Docker, sur ghcr.io/t3hx/<projet>:latest.
Secrets et variables à poser dans chaque repo (un compte perso n'a pas de secrets partagés entre repos) :
Nom
Type
Valeur
TS_OAUTH_CLIENT_ID
secret
client OAuth Tailscale (tag:ci)
TS_OAUTH_SECRET
secret
idem
DOKPLOY_PROD_WEBHOOK_URL
secret
Dokploy → app → onglet Deployments → Webhook URL
PROD_URL
variable
https://<domaine public>
VPS_TAILSCALE_IP
variable
IP Tailscale du VPS (vérification à l'origine, sans Cloudflare)
En ligne de commande, depuis le dossier du repo :
gh secret set DOKPLOY_PROD_WEBHOOK_URL
gh variable set PROD_URL --body "https://solarsys.nspace.link"
gh variable set VPS_TAILSCALE_IP --body "100.x.y.z"
Où en sont tes trois repos :
Repo
Dockerfile + version.json
Déploiement actuel
Ce qu'il manque
solarsys-react-app
oui
webhook + vérif version.json
rien
myPortfolio
Dockerfile oui, version.json non
API Dokploy + vérif sourceType
version.json, secret webhook, 2 variables
owlog-nuxt-app
non
aucun via la CI
Dockerfile (plus tard)
