# pterodactyl-webapp-egg

Egg Pterodactyl tout-en-un pour héberger **un site statique** (HTML/CSS/JS) avec **un backend Node.js/Express optionnel**. Express préinstallé, dépendances additionnelles auto-installées au démarrage.

## Fonctionnalités

- **Site statique servi automatiquement** depuis `public/`
- **Backend Node.js optionnel** via un simple `routes.js`
- **Express préinstallé** + npm install auto si tu modifies `package.json`
- **Fallback SPA** intégré (compatible React/Vue Router)
- **Page de démo** créée à l'install si `public/` est vide
- **Port automatique** depuis l'allocation Pterodactyl (`process.env.SERVER_PORT`)

## Installation de l'egg

1. Télécharger [`egg-webapp.json`](./egg-webapp.json)
2. Dans le panel Pterodactyl : **Admin → Nests → Import Egg**
3. Sélectionner le fichier JSON et l'importer

## Création d'un serveur

1. Créer un nouveau serveur avec l'egg **WebApp (Static + Node.js)**
2. Choisir Node.js 20 ou 22 dans Docker Image
3. Allouer un port
4. Démarrer → Express et les fichiers de base s'installent automatiquement

## Cas d'usage

### Cas 1 — Site statique seul

Uploade tes fichiers dans `public/` :
