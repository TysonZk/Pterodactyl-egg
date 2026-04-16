# pterodactyl-nginx-static-egg

Egg Pterodactyl pour héberger des **sites statiques HTML/CSS/JS** avec un **nginx compilé en statique** (zéro dépendance au runtime). Conçu pour fonctionner derrière un reverse proxy nginx + Let's Encrypt côté VPS.

## Fonctionnalités

- **Nginx compilé from source** avec PCRE statique → aucune dépendance externe au runtime
- **Port automatique** depuis l'allocation Pterodactyl
- **Aucune variable** à configurer
- **Cache + gzip** activés par défaut
- **Image runtime ultra-légère** : `ghcr.io/parkervcp/yolks:debian`

## Installation de l'egg

1. Télécharger [`egg-nginx-static.json`](./egg-nginx-static.json)
2. Dans le panel Pterodactyl : **Admin → Nests → Import Egg**
3. Sélectionner le fichier JSON et l'importer

## Création d'un serveur

1. Créer un nouveau serveur avec l'egg **Nginx Static**
2. Allouer un port (ex : `27010`)
3. Démarrer le serveur — la première install compile nginx (~3 min)
4. Déposer les fichiers dans `webroot/` via le File Manager ou SFTP

Le site est ensuite accessible sur `http://<IP>:<PORT>`.

## ----------OPTIONNEL-----------

## Architecture recommandée (avec SSL et blocage IP)

```
Visiteur
   ↓
Cloudflare DNS (DNS only, nuage gris)
   ↓
VPS:443 → nginx reverse proxy + SSL Let's Encrypt
   ↓
Container Pterodactyl:PORT (Wings/Docker)
   ↓
nginx compilé (egg)
   ↓
webroot/
```

L'accès direct via `IP:PORT` est bloqué au niveau **iptables** (chaîne `DOCKER-USER`).

### Configuration du VPS

#### 1. Vhost `default-block` (à faire une seule fois)

Crée un vhost qui bloque tous les accès directs par IP, sauf les challenges ACME pour les futurs certbot :

```nginx
# /etc/nginx/sites-available/default-block
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 444;
    }
}
```

```bash
mkdir -p /var/www/html
ln -s /etc/nginx/sites-available/default-block /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

#### 2. Déployer un nouveau site

```bash
DOMAIN="monsite.exemple.com"
PORT="27010"
NODE_IP="51.178.19.2"

cat > /etc/nginx/sites-available/$DOMAIN <<EOF
server {
    listen 80;
    server_name $DOMAIN;
    location / {
        proxy_pass http://$NODE_IP:$PORT;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
EOF

ln -s /etc/nginx/sites-available/$DOMAIN /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

certbot --nginx -d $DOMAIN

iptables -I DOCKER-USER -p tcp --dport $PORT -j DROP
netfilter-persistent save
```

#### 3. Supprimer un site

```bash
DOMAIN="monsite.exemple.com"
PORT="27010"

rm /etc/nginx/sites-enabled/$DOMAIN
rm /etc/nginx/sites-available/$DOMAIN
certbot delete --cert-name $DOMAIN
iptables -D DOCKER-USER -p tcp --dport $PORT -j DROP
netfilter-persistent save
nginx -t && systemctl reload nginx
```

## Dépôt des fichiers

| Méthode | Comment |
| :--- | :--- |
| File Manager Pterodactyl | Glisser-déposer dans `webroot/` |
| SFTP | Credentials SFTP du serveur, dossier `webroot/` |
| Build local (Vite/React/Vue) | `npm run build` puis upload du contenu de `dist/` |

`index.html` doit être à la racine de `webroot/`.

## Chemins importants

| Élément | Chemin |
| :--- | :--- |
| Fichiers du site | `/home/container/webroot/` |
| Binaire nginx compilé | `/home/container/bin/nginx` |
| Logs | `/home/container/logs/` |
| Config nginx (générée au runtime) | `/tmp/nginx.conf` |

## Troubleshooting

**502 Bad Gateway** : le serveur Pterodactyl n'est pas démarré ou `proxy_pass` pointe sur la mauvaise IP. Vérifier `ss -tlnp | grep PORT` sur le VPS.

**Certbot échoue (empty reply)** : le `default-block` capture les challenges. Vérifier que l'exception `/.well-known/acme-challenge/` est bien présente.

**"Address already in use"** : un vhost nginx du VPS bind un port utilisé par Wings. Vérifier `nginx -T | grep listen`.

**Le site reste accessible par IP** : la règle iptables doit être dans `DOCKER-USER`, pas `INPUT` (Docker contourne `INPUT`).

## Limitations

- **Sites statiques uniquement** — pas de PHP, pas de Node.js, pas de SSR
- Pas de gestion SSL **dans** le container (Wings n'expose pas les ports 80/443 directement, le SSL se gère sur le reverse proxy du VPS)

Pour du Node.js/Next.js SSR, utiliser un egg Node.js dédié.

## Licence

MIT
