# Tunnels frp + Caddy

Cette stack expose un service local sur `https://prenom.tunnel.<DOMAINE>`.
frps gère les tunnels et Caddy termine le TLS avec un certificat wildcard OVH.
Traefik ne fait que le passthrough TCP TLS ; aucun domaine ni resolver Traefik
n'est nécessaire dans Dokploy.

## Prérequis

- Un serveur Dokploy avec Docker Compose et le réseau externe `dokploy-network`.
- Dans la zone DNS OVH, deux enregistrements A vers l'IP du serveur :
  `tunnel` et `*.tunnel`.
- Un token API OVH créé sur <https://api.ovh.com/createToken/> avec l'endpoint
  `ovh-eu` et les droits `GET`, `POST`, `DELETE` sur `/domain/zone/*`.
- Le port TCP `7000` ouvert dans le firewall du serveur.

## Déploiement dans Dokploy

1. Créez une application **Compose** liée au dépôt et choisissez
   `tunnels/docker-compose.yml` comme fichier Compose. Utilisez le mode
   **Docker Compose**, pas le mode Stack.
2. Ajoutez dans les variables d'environnement Dokploy :
   `TUNNEL_DOMAIN`, `OVH_ENDPOINT=ovh-eu`,
   `OVH_APPLICATION_KEY`, `OVH_APPLICATION_SECRET`, `OVH_CONSUMER_KEY` et
   `FRP_TOKEN`. Marquez les quatre secrets (les trois clés OVH et le token frp)
   comme secrets si votre version de Dokploy le permet.
3. Déployez. Le Compose monte `Caddyfile` et `frps.toml` avec `configs` ; aucun
   montage `../files/...` ni domaine dans l'interface Dokploy n'est requis.

Au premier démarrage, le build custom de Caddy puis le challenge DNS et
l'émission du wildcard prennent généralement 1 à 3 minutes. Suivez les logs
des services `caddy` et `frps` dans Dokploy. Côté Caddy, attendez notamment le
message indiquant que le certificat a été obtenu ; côté frps, vérifiez que les
ports `7000` et `8080` sont à l'écoute.

Les labels utilisent la syntaxe Traefik v3 :
`HostSNIRegexp(\`^.+\\.tunnel\\.<DOMAINE>$\`)`. Si votre Dokploy utilise
Traefik v2, remplacez le label `rule` par :
`HostSNIRegexp(\`{sub:[a-z0-9-]+}.tunnel.<DOMAINE>\`)`.

## Connexion d'un développeur

1. Installez `frpc` v0.71.x (la version doit correspondre à frps).
2. Copiez `frpc.example.toml` en `frpc.toml`, remplacez `example.com`,
   `localPort` et `subdomain` (`prenom`), puis exportez le même token :

   ```sh
   export FRP_TOKEN='le-token-fourni-par-lequipe'
   frpc -c frpc.toml
   ```

3. Ouvrez `https://prenom.tunnel.example.com`. Le proxy HTTP de frp conserve
   le Host et les WebSockets ; SSR et HMR passent donc par la même URL.

## Dépannage

- **Certificat absent** : vérifiez les deux DNS, `OVH_ENDPOINT=ovh-eu`, les
  trois clés OVH, les droits `/domain/zone/*` et les logs Caddy. Vérifiez aussi
  que le port 443 public arrive bien sur Traefik.
- **Connexion frpc refusée** : vérifiez que le port TCP 7000 est ouvert, que
  `serverAddr` est `tunnel.<DOMAINE>` et que le token est identique.
- **404 ou `no route found`** : vérifiez `subdomain`, `subDomainHost`, le nom
  DNS demandé et que frpc est connecté avant le test HTTP.
- **502** : vérifiez que frps est démarré et que Caddy peut joindre
  `frps:8080` sur le réseau `internal`.
