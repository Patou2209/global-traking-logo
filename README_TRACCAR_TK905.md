📍 TK905 Tracking Platform – Setup Guide
🎯 Objectif

Mettre en place un serveur Traccar sur un VPS Ubuntu avec :

IP publique déjà disponible

Docker + Docker Compose

PostgreSQL

Support du TK905 4G (protocole Huabao / JT808)

Device visible et en ligne dans Traccar

# 🖥️ 1. Préparation du VPS (Ubuntu)

# Connexion au serveur :
ssh root@VOTRE_IP_PUBLIQUE
Mise à jour système :
apt update && apt upgrade -y
Installer Docker :
apt install -y docker.io docker-compose
systemctl enable docker
systemctl start docker
Vérifier :
docker --version
docker compose version

# 📁 2. Création de l’arborescence projet
mkdir -p /opt/tk905-platform/infra/traccar
cd /opt/tk905-platform

# 🐘 3. docker-compose.yml

# Créer le fichier :
nano docker-compose.yml
# Contenu :
version: "3.9"

services:

  postgres:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_USER: tk905
      POSTGRES_PASSWORD: tk905_password
      POSTGRES_DB: tk905_db
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  traccar:
    image: traccar/traccar:latest
    restart: always
    depends_on:
      - postgres
    ports:
      - "8082:8082"
      - "5015:5015"
      - "5015:5015/udp"
      - "5023:5023"
      - "5023:5023/udp"
    volumes:
      - ./infra/traccar/traccar.xml:/opt/traccar/conf/traccar.xml

volumes:
  pgdata:

⚙️ 4. Configuration traccar.xml

# Créer :
nano /opt/tk905-platform/infra/traccar/traccar.xml

# Contenu minimal fonctionnel :

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
  <entry key='database.driver'>org.postgresql.Driver</entry>
  <entry key='database.url'>jdbc:postgresql://postgres:5432/tk905_db</entry>
  <entry key='database.user'>tk905</entry>
  <entry key='database.password'>tk905_password</entry>

  <!-- Port JT808 -->
  <entry key='jt808.port'>5023</entry>

  <!-- Port Huabao (utilisé par ton TK905 4G) -->
  <entry key='huabao.port'>5015</entry>
</properties>

🔥 5. Ouvrir le firewall (UFW)
ufw allow 8082/tcp
ufw allow 5015/tcp
ufw allow 5015/udp
ufw allow 5023/tcp
ufw allow 5023/udp

🚀 6. Lancer la plateforme

# Depuis :
cd /opt/tk905-platform

# Démarrer :
docker compose up -d

# Rédemarer traccar
docker compose restart traccar

# Voir les ports Ouvert
ss -lntp | grep -E ":8082|:5015|:5023"

# Voir les logs Traccar
docker logs -f tk905-platform-traccar-1

# Vérifier :
docker ps

🌐 7. Accéder à Traccar

# Dans le navigateur :
http://VOTRE_IP_PUBLIQUE:8082

Créer le premier utilisateur administrateur.

📱 8. Ajouter le Device dans Traccar

# Dans l’interface :

Devices → Add Device

Name : TK905

Unique ID : 009590053841
⚠️ Important : mettre les zéros devant

Enregistrer.

📡 9. Configurer le tracker TK905

# Envoyer par SMS au tracker :

- admin123456 numeroTelephone
- apn123456 vodanet(ou autre reseau)
- gprs123456
- adminip123456 VOTRE_IP_PUBLIQUE 5015
- upload123456 30
- sleep123456 off
- timezone123456 1

📜 10. Vérifier que les données arrivent

# Voir les logs en direct :
docker exec -it tk905-platform-traccar-1 sh -lc "tail -f /opt/traccar/logs/tracker-server.log"

🧪 11. Vérifier via API REST

# Lister les devices :
curl -u "EMAIL:PASSWORD" http://127.0.0.1:8082/api/devices

# RESTRICTION DES FONCTIONNALITES AUX NON ADMINS
Absolument, c'est une configuration classique pour transformer Traccar en une plateforme commerciale ou privée contrôlée.
En tant qu'Administrateur, vous avez deux leviers pour verrouiller cela :
1. Désactiver l'ajout d'appareils pour UN utilisateur précis
Si vous voulez que certains utilisateurs puissent ajouter des appareils et d'autres non :
Allez dans Réglages > Utilisateurs.
Sélectionnez l'utilisateur concerné et cliquez sur l'icône Éditer (le crayon).
Cochez la case Lecture seule (Read-only).
Résultat : L'utilisateur peut voir ses traceurs, mais le bouton "+" pour ajouter un appareil disparaît de son interface.
Alternativement, cochez Désactivé (Disabled) sous la section "Attributs" pour le champ "Device Limit" et mettez la valeur à 0.
2. Désactiver l'ajout d'appareils pour TOUS les utilisateurs (Global)
Si vous voulez être le seul à pouvoir enregistrer des terminaux sur votre serveur :
Allez dans Réglages > Serveur.
Décochez la case Attribution d'appareils (Device Read Only).
Note : Dans les versions récentes, cela se trouve souvent dans les "Attributs" du serveur. Ajoutez l'attribut readonly ou positionnez Device Limit à 0 par défaut pour les nouveaux comptes.
3. Comment l'Admin "attache" le device ?
Une fois que l'utilisateur est restreint, voici votre flux de travail :
L'Admin crée le device (nom, IMEI du TK905).
L'Admin va dans Réglages > Utilisateurs.
Sélectionne l'utilisateur > bouton Connexions (icône chaîne).
Coche le device créé pour le "lier" à ce compte.
L'appareil apparaîtra instantanément sur la carte de l'utilisateur.
Pourquoi est-ce utile pour votre Gateway ?
Si vous utilisez Traccar pour parser les données et les renvoyer vers une IP publique, limiter les utilisateurs évite qu'ils ne modifient les identifiants (IMEI), ce qui casserait votre chaîne de transmission de données.
Souhaitez-vous que je vous montre comment personnaliser l'interface (logo, titre) pour que vos utilisateurs ne voient pas la marque "Traccar" ?



Pour transformer votre instance Traccar en une plateforme à votre propre image (White Label), vous devez modifier les Attributs du Serveur.
Voici la marche à suivre pour remplacer le logo et le nom sans toucher au code source :
1. Préparer vos images
Hébergez votre logo sur un serveur ou dans un dossier accessible par votre VPS. Vous aurez besoin de deux liens (URL) :
Logo de connexion : Grande image pour la page de login.
Logo de l'interface : Petite image (icône) pour le coin supérieur de la carte.
2. Configurer via l'interface Admin
Connectez-vous avec votre compte Administrateur.
Allez dans Réglages (icône roue dentée) > Serveur.
Cherchez la section Attributs et cliquez sur le bouton + (Ajouter).
Ajoutez les attributs suivants un par un :
Nom de l'attribut	Valeur (Exemple)	Effet
title	Mon GPS Kinshasa	Remplace "Traccar" dans l'onglet du navigateur.
logo	https://votre-site.com	Affiche votre logo sur la page de connexion.
logoInverted	https://votre-site.com	Affiche votre logo sur l'interface de suivi (fond sombre).
3. Cacher les liens officiels
Pour que les utilisateurs ne puissent pas cliquer sur "Traccar" ou s'enregistrer eux-mêmes :
Ajoutez l'attribut registration avec la valeur false (désactive le bouton "S'enregistrer").
Ajoutez l'attribut ui.disableDocumentation avec la valeur true (cache les liens d'aide officiels).
4. Personnalisation avancée (Docker)
Si vous voulez aller plus loin et changer les couleurs ou le texte "Powered by Traccar", vous devez modifier les fichiers dans le dossier /opt/traccar/web/.
Le fichier index.html contient les textes de base.
Le fichier l10n/fr.json vous permet de traduire ou renommer n'importe quel terme de l'interface (ex: remplacer "Appareil" par "Véhicule").
Rappel Important pour votre Gateway
Ces modifications sont visuelles. Elles n'affectent pas le Parsing des données de votre TK905 ni le Forwarding (Gateway) vers votre IP publique.


# AUTHENTIFICATIONS
Digital Ocean Passphrsase : patou 
Name : patoukaziama@gail.com
Sign in methode : github (Patou2209) professionalgith@gmail.com
Public IP : 157.245.181.252
Connexion au serveur : ssh root@157.245.181.252

Traccar serveur :
Connexion au compte admin : http://157.245.181.252:8082
Admin Principal : 
Nom : TECHNOWEB
Email : patoukaziama@gmail.com
Password : patou2209
Admin pour Tous devices : 
Nom : tk905-api
Email : technoweb2209@gmail.com
Password : patou2209

# FAIRE DU BRANDING
1 — Appliquer le branding correctement via l’API Traccar

📍 À exécuter sur le VPS (SSH), dans n’importe quel dossier.

Remplace le mot de passe si nécessaire.
BASH:

set -e

TRACCAR_URL="http://127.0.0.1:8082"
TRACCAR_USER="patoukaziama@gmail.com"
TRACCAR_PASS="patou2209"

LOGO_URL="https://patou2209.github.io/global-traking-logo/logo.png"

echo "==> 1) Update server attributes (title + logo + logoInverted + disable doc)..."

curl -s -u "${TRACCAR_USER}:${TRACCAR_PASS}" \
  -H "Content-Type: application/json" \
  -X PUT "${TRACCAR_URL}/api/server" \
  -d @- <<JSON
{
  "id": 1,
  "attributes": {
    "timezone": "Africa/Kinshasa",
    "title": "GLOBAL TRAKING",
    "logo": "${LOGO_URL}",
    "logoInverted": "${LOGO_URL}",
    "ui.disableDocumentation": true
  },
  "registration": false,
  "forceSettings": true
}
JSON

echo
echo "==> 2) Lire la config serveur pour vérifier..."
curl -s -u "${TRACCAR_USER}:${TRACCAR_PASS}" "${TRACCAR_URL}/api/server" | head -c 2000
echo
echo
echo "✅ Done. Ouvre l’UI puis fais CTRL+F5 (ou navigation privée)."

|||||||||||||||||||||||||||| fin bash

# Agrandir le logo

docker exec -it tk905-platform-traccar-1 sh -lc '
WEB=/opt/traccar/web

cat > "$WEB/custom.css" << "CSS"

/* ===========================
   1️⃣ Remettre le drapeau normal
=========================== */
header img {
  height: 24px !important;
  width: auto !important;
  max-height: 24px !important;
  max-width: none !important;
}

/* ===========================
   2️⃣ Agrandir UNIQUEMENT ton logo
=========================== */
img[src*="patou2209.github.io/global-traking-logo"] {
  width: 360px !important;   /* Ajuste ici si tu veux plus grand */
  height: auto !important;
  max-width: none !important;
  max-height: none !important;
  display: block;
  margin: 0 auto 15px auto;
}

CSS

echo "Patch appliqué"
'
docker restart tk905-platform-traccar-1

# checker le stockage VPS:

- sur le vps: htop

🟢 3️⃣ Comment upgrader ton VPS (DigitalOcean)

👉 Tu ne touches à AUCUN fichier.
👉 Tu ne perds PAS tes données.
👉 C’est quasi instantané.

🔹 Étapes dans DigitalOcean

Connecte-toi sur :
https://cloud.digitalocean.com

Clique sur ton Droplet (ton VPS)

Clique sur Resize

Choisis :
2 vCPU
4GB RAM

Choisis :

CPU optimized ou Regular (Regular suffit pour commencer)

Active “Increase disk size” si tu veux plus de stockage

Clique Resize Droplet

Attends 1–2 minutes

Redémarre le droplet si demandé

après upgrade on fait sur vps: reboot
puis on relance a nouveau le vps


