# Installer stillfox sur son propre serveur

stillfox se compose de deux conteneurs : le **serveur** (la base de données, l'API et la passerelle qui parle à votre
centrale HC2 sur le port 8888) et le **site** (l'interface web). Ce guide les installe à partir des images publiques et
des fichiers joints à chaque release. Les exemples utilisent `exemple.fr` pour le nom de domaine, `192.0.2.10` pour
l'adresse de votre serveur et `203.0.113.7` pour l'adresse publique de votre domicile : remplacez-les par les vôtres.

Il n'y a **rien à remplir avant le premier démarrage** : le serveur tire lui-même ses clés secrètes, puis un écran
d'installation (protégé par un code écrit dans le journal du serveur) crée votre compte et vous demande l'adresse publique
du site. Deux façons de procéder : **le script** (section « En trois commandes »), ou **à la main** (sections 1 à 9).

Les images publiques :

- `ghcr.io/ppollet73/stillfox-serveur:<version>`
- `ghcr.io/ppollet73/stillfox-site:<version>`

Chaque version est aussi désignée par l'empreinte sha256 (digest) de ses images : le `docker-compose.yml` joint à la
release écrit `image: ghcr.io/ppollet73/stillfox-serveur@sha256:…` (et de même pour le site). Il ne peut donc tirer que les
images publiées avec cette release, même si une étiquette de version était déplacée plus tard. Il n'y a pas de numéro de
version à renseigner : changer de version, c'est remplacer ce fichier (section 9).

Chaque release joint quatre fichiers : `docker-compose.yml`, `.env.example`, `stillfox.sh` (le script) et
`empreintes.sha256`. Le `docker-compose.yml` est générique : il ne construit rien (il tire les images publiques), ne
demande aucune variable et ne fixe que le sous-réseau interne de ses deux conteneurs.

## En trois commandes (le script)

Le script `stillfox.sh` fait l'installation, la mise à jour et le retour arrière. Il se lance **sur le serveur** (en SSH)
ou, pour un essai sur votre PC, dans le terminal de Linux ou de macOS ; sous Windows, dans WSL (Docker Desktop en dépend
déjà). Il a besoin de Docker, de `curl` et de `bash` 4 ou plus, et se lance en **root** (`sudo`). Il n'y a pas de
`curl … | sh` : on télécharge, on contrôle l'empreinte, puis on lance.

```sh
curl -fsSL --remote-name-all https://github.com/ppollet73/stillfox/releases/latest/download/stillfox.sh https://github.com/ppollet73/stillfox/releases/latest/download/empreintes.sha256
grep ' stillfox.sh$' empreintes.sha256 | sha256sum -c
sudo bash stillfox.sh installer
```

La deuxième commande doit afficher `stillfox.sh: OK` ; sinon n'allez pas plus loin. (Sous macOS, `shasum -a 256 -c` à la place
de `sha256sum -c`.) Le script pose alors ses questions, ou prend des options : `--mode serveur` (nom de domaine, HTTPS par Caddy)
ou `--mode local` (Docker de ce PC, site sur `http://localhost`), `--domaine exemple.fr`, `--dossier`, `--version vX.Y.Z`,
`--oui` (sans question).

Ce que le script fait, dans l'ordre, **sans rien écrire tant que les contrôles ne sont pas réussis** : contrôle de la machine
(Docker et `docker compose` v2, 1 Go de mémoire, 2 Go de disque, ports libres, dossier absent ou vide), téléchargement du
`docker-compose.yml` de la dernière release et contrôle de son empreinte, tirage des deux images et contrôle de leur
empreinte, arborescence (sections 2 et 3), puis démarrage et attente de la santé des conteneurs. En mode `serveur`, il
vérifie aussi que `exemple.fr` et `api.exemple.fr` pointent vers l'adresse publique de la machine, et installe Caddy
(section 6). À la fin il affiche l'adresse de l'écran d'installation et le **code d'installation** (à usage unique).

Ce qu'il **ne fait pas** : il ne touche ni au pare-feu ni à la configuration du système, n'installe pas Docker, ne supprime
jamais de données. Il vous rappelle les étapes qui restent à vous : copier la clé des sauvegardes hors du serveur, importer
vos données depuis l'écran du site, rattacher et rediriger la centrale, le pare-feu (section 8).

Ensuite :

```sh
sudo bash stillfox.sh mettre-a-jour     # vers la dernière release (ou --version vX.Y.Z)
sudo bash stillfox.sh retour            # revient à la version précédente et à sa copie de la base
```

`mettre-a-jour` contrôle tout **avant** de couper quoi que ce soit (empreintes, images, espace disque), affiche les notes des
versions à installer, copie la base serveur arrêté (`data/avant-<version>/`, les deux dernières copies sont gardées), démarre
la nouvelle version et, si les conteneurs ne sont pas sains au bout de 3 minutes, revient seul à l'ancienne version. Si les
notes portent une section **« À faire avant la mise à jour »** (une modification du `Caddyfile`, par exemple), il s'arrête
sans rien couper : faites ce qu'elle demande, puis relancez avec `--gestes-faits`. `retour` annonce que les données écrites
depuis la mise à jour seront perdues et garde la base du moment à côté (`data/ecartee-<version>/`), jamais supprimée. Il ne fait
rien si la copie de la base manque. Le script garde son état dans `.stillfox-etat` ; ne l'éditez pas.

Les sections suivantes décrivent la même installation **à la main**, pour qui ne veut pas du script, et servent de référence.

## 1. Prérequis

- Un serveur Linux (un petit VPS suffit : le serveur est borné à 384 Mo de mémoire et le site à 192 Mo), avec **une adresse
  IPv4 publique fixe**.
- Docker et Docker Compose (le plugin `docker compose`).
- Un nom de domaine, avec deux noms qui pointent vers le serveur : `exemple.fr` (le site) et `api.exemple.fr` (l'API).
- Les ports **80 et 443** du serveur ouverts sur Internet et libres (aucun autre serveur web) : **Caddy**, installé à la
  section 6 dans le même compose, s'y place devant le site et l'API et obtient seul les certificats HTTPS. Le site
  refuse de fonctionner sans HTTPS (sauf en local, sur `localhost`) : le cookie de session est « Secure ». Si votre serveur a
  déjà un proxy inverse, gardez-le et reprenez seulement le contenu du `Caddyfile` de la section 6.
- Un compte myfox.me qui fonctionne encore : l'import de vos données s'y connecte. **Après le 31 mars 2027, il ne sera plus
  possible de les récupérer.**
- **Vos propres identifiants de client de l'API Myfox** (`MYFOX_CLIENT_ID` et `MYFOX_CLIENT_SECRET`), obtenus sur le
  portail développeur de myfox.me, tant qu'il existe. Chaque installateur utilise les siens : ceux du projet ne sont jamais
  publiés. Ils ne servent qu'à l'import (étape 7, puis « Importer depuis myfox.me » sur le site) ; vous ne les écrivez pas dans `.env`.
  Somfy n'accepte une application personnelle qu'avec le compte myfox.me de son créateur : créez la vôtre avec le compte
  dont vous importez les données (une application par compte, créée d'un clic sur le portail : « Client ID », « Client Secret »,
  « Status : Active »). Somfy la désactive sans préavis si elle demande trop de jetons : un import n'en demande qu'un.
- Facultatif : un relais SMTP (inscription et mot de passe oublié par e-mail), un compte Pushover (notifications mobiles).

## 2. Arborescence

Tout vit dans `/opt/stillfox`. Créez-la en root :

```sh
install -d -m 755 /opt/stillfox
install -d -o 10001 -g 10001 -m 700 /opt/stillfox/data
install -d -m 700 /opt/stillfox/secrets
install -d -o 10001 -g 10001 -m 700 /opt/stillfox/secrets/generes
install -d -m 700 /opt/stillfox/caddy /opt/stillfox/caddy/data /opt/stillfox/caddy/config
```

```
/opt/stillfox/
├── data/                       uid 10001, droits 700 : base, journaux, sauvegardes, paquets d'import
├── secrets/                    root, droits 700
│   ├── cle-sauvegardes         uid 10001, droits 400 (section 3)
│   └── generes/                uid 10001, droits 700 : clés tirées par le serveur au premier démarrage
├── caddy/                      root, droits 700 : certificats HTTPS (data) et état de Caddy (config)
├── Caddyfile                   root, droits 644 (section 6)
├── docker-compose.yml          root, droits 644 (fichier joint à la release)
├── docker-compose.override.yml root, droits 644 : le service Caddy (section 6)
└── .env                        root, droits 600, facultatif (section 5)
```

Le serveur tourne sous l'utilisateur **10001** dans le conteneur, comme le site : un dossier de données qui appartient à
root, ou une clé en 600 appartenant à root, lui seraient illisibles. Les dossiers `data` et `secrets/generes` doivent exister
**avant** le premier démarrage ; Docker ne les crée pas. Le site lit, en lecture seule, le secret que le serveur écrit dans
`secrets/generes` : les clés elles-mêmes (secret partagé entre le site et l'API, clé des paquets d'import, clé des réglages)
sont tirées au hasard par le serveur au premier démarrage et relues ensuite ; vous n'avez rien à créer ni à écrire.

## 3. La clé des sauvegardes

Le serveur chiffre ses sauvegardes quotidiennes avec une clé de 64 caractères hexadécimaux. C'est la seule clé que vous créez
(une version suivante la tirera aussi toute seule) :

```sh
openssl rand -hex 32 > /opt/stillfox/secrets/cle-sauvegardes
chown 10001:10001 /opt/stillfox/secrets/cle-sauvegardes
chmod 400 /opt/stillfox/secrets/cle-sauvegardes
```

**Gardez une copie de ce fichier hors du serveur** (gestionnaire de mots de passe, autre machine). Sans elle, vos sauvegardes
sont illisibles. Les clés de `secrets/generes` vivent dans le dossier d'installation : sauvegardez-le aussi (section 10).

## 4. Télécharger les fichiers et contrôler les empreintes

Depuis la page de la release choisie, téléchargez `docker-compose.yml` et `empreintes.sha256` dans `/opt/stillfox` (adaptez
l'adresse à la version voulue). Puis contrôlez le compose :

```sh
cd /opt/stillfox
grep ' docker-compose.yml$' empreintes.sha256 | sha256sum -c
```

La ligne doit afficher `OK`. En échec : ne continuez pas, retéléchargez le fichier. (Les autres lignes de `empreintes.sha256`
donnent les empreintes des autres fichiers et des images ; ce ne sont pas des fichiers à contrôler ici, d'où le `grep`.)

## 5. Le fichier `.env` (facultatif)

**Aucune variable n'est obligatoire.** Un fichier `.env` à côté du compose ne sert qu'à régler ce qui sort de l'ordinaire.
Compose le lit seul ; il doit être en UTF-8 **sans BOM**, avec des fins de ligne **LF**, en droits 600, et ne se publie ni ne
s'affiche jamais. Les variables utiles :

| Variable | Rôle |
|---|---|
| `HC2_TRUSTED_PROXIES` | adresse du proxy inverse devant le site : **obligatoire avec le Caddy de la section 6** (voir plus bas) |
| `MYFOX_API_TRUSTED_PEERS` | pairs autorisés à annoncer l'adresse de leur visiteur : le site, puis le proxy (voir plus bas) |
| `VPS_IPV4` | adresse IPv4 publique du serveur, si vous voulez que le port 8888 n'écoute que sur elle (par défaut : toutes les interfaces) |
| `MYFOX_SITE_URL`, `HC2_PUBLIC_ORIGIN`, `HC2_PUBLIC_API_URL` | adresses publiques : **à ne pas renseigner**, elles se saisissent à l'écran d'installation ; une valeur posée ici l'emporte sur l'écran |
| `RUST_LOG` | niveau des journaux (`info` par défaut) |

Pour réparer sur le serveur un fichier écrit depuis Windows (BOM, fins de ligne) :

```sh
sed -i '1s/^\xEF\xBB\xBF//; s/\r$//' /opt/stillfox/.env
```

## 6. Caddy, le proxy inverse HTTPS

Le site (port 8383) et l'API (port 8480) ne sont jamais ouverts à Internet : le `docker-compose.yml` joint les publie
**sur `127.0.0.1` seulement**. C'est Caddy qui reçoit les visiteurs sur les ports 80 et 443, obtient et renouvelle seul
les certificats HTTPS (Let's Encrypt), puis transmet au site et à l'API. Il tourne dans un conteneur en réseau « host »
pour les atteindre à `127.0.0.1:8383` et `127.0.0.1:8480` : ni sous-réseau ni adresse fixe à prévoir de votre côté.

Avant de le démarrer, les deux noms `exemple.fr` et `api.exemple.fr` doivent déjà pointer vers le serveur et les ports 80
et 443 être ouverts : sinon Let's Encrypt ne peut pas valider le domaine et Caddy reste sans certificat (il réessaie seul).

### Le service Caddy

Créez `/opt/stillfox/docker-compose.override.yml`. Docker Compose le lit automatiquement avec `docker-compose.yml` :
`docker compose up -d` démarre alors les trois conteneurs, et la mise à jour de la section 9, qui ne remplace que
`docker-compose.yml`, ne le touche pas.

```yaml
services:
  caddy:
    image: caddy:2.11.9@sha256:e40b94d0a43b5ff69cb20568d73f10c70703df7b0ffc2298eb3d723cdc9d0bd1   # épinglée par empreinte
    restart: unless-stopped
    network_mode: host          # écoute sur 80 et 443, joint le site et l'API sur 127.0.0.1
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE]
    read_only: true
    security_opt: ["no-new-privileges:true"]
    tmpfs: [/tmp]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/data:/data      # certificats : à garder d'une version à l'autre
      - ./caddy/config:/config
    mem_limit: 128m
    memswap_limit: 128m
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
```

### Le `Caddyfile`

Créez `/opt/stillfox/Caddyfile` en remplaçant `exemple.fr` par votre domaine :

```caddy
exemple.fr {
	encode zstd gzip
	header Strict-Transport-Security "max-age=31536000"
	reverse_proxy 127.0.0.1:8383 {
		header_up -X-Myfox-Visitor-Address
	}
}

api.exemple.fr {
	encode zstd gzip
	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		Referrer-Policy "no-referrer"
	}
	reverse_proxy 127.0.0.1:8480 {
		header_up X-Myfox-Visitor-Address {client_ip}
	}
}

# La centrale HC2 appelle ce nom en HTTP simple au démarrage ; elle n'attend que cette réponse fixe.
http://hc2.steadyfox.com {
	handle /box/btf/* {
		respond ":)[0]
" 200
	}
}
```

Si vous restreignez l'accès au site à quelques adresses (recommandé tant que vous êtes seul), faites-le dans ce fichier.

Validez le fichier, puis, Caddy une fois démarré (étape 7), rechargez-le après chaque modification, sans coupure :

```sh
cd /opt/stillfox
docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Caddy ne démarre qu'avec les deux autres conteneurs (`docker compose up -d`, étape 7). Contrôle :
`docker compose logs caddy | grep -i certificate` doit montrer « certificate obtained successfully » pour chacun des
deux noms, puis `https://exemple.fr` s'ouvre avec un cadenas.

### Adresse réelle du visiteur (obligatoire derrière un proxy inverse)

Seul un pair déclaré a le droit d'annoncer l'adresse de son visiteur (adresses seules, pas de réseau). Le
`docker-compose.yml` joint fixe le sous-réseau de son réseau interne (section `networks`, champs `subnet` et `gateway`)
et l'adresse du site (champ `ipv4_address` du service `site`), puis déclare cette adresse par défaut dans
`MYFOX_API_TRUSTED_PEERS` : l'API fait donc confiance au site sans réglage de votre part. Si ce sous-réseau est déjà
pris sur votre machine, changez-le dans le fichier, avec les adresses qui en dépendent.

**`HC2_TRUSTED_PROXIES` est en revanche obligatoire dès qu'un proxy inverse (Caddy, section 6) est devant le site.**
Laissée vide, tous les visiteurs apparaissent avec l'adresse du proxy : un seul visiteur qui multiplie les échecs de
connexion ou d'inscription bloque alors tout le monde. Avec le Caddy de la section 6 (`network_mode: host`), le site et
l'API voient la passerelle du réseau du compose (champ `gateway`) : dans `.env`, mettez cette adresse dans
`HC2_TRUSTED_PROXIES`, et `MYFOX_API_TRUSTED_PEERS=<adresse du site>,<adresse de la passerelle>` (le site, puis le proxy
pour le bloc de l'API, qui envoie lui-même l'adresse du visiteur). Contrôle : après quelques connexions de deux appareils
différents, les journaux du site doivent montrer deux adresses distinctes, pas celle de la passerelle.
Si votre proxy a une autre adresse (Caddy sur le réseau du compose avec une adresse fixe, par exemple), déclarez celle-là.
Sans proxy inverse (essai en local), ne renseignez rien. (Le script `stillfox.sh installer` écrit ces deux lignes lui-même.)

## 7. Premier démarrage, écran d'installation, import

Un serveur dont la base est vide **démarre** : c'est le début d'une installation. Tirez les deux images, contrôlez leur
empreinte, puis démarrez. Le fichier compose joint désigne chaque image par son empreinte sha256 : Docker refuse déjà une
image dont le contenu ne correspond pas. Le contrôle ci-dessous le vérifie en plus, contre `empreintes.sha256` (l'empreinte
que Docker a retenue pour chaque image est la ligne `RepoDigests`) :

```sh
cd /opt/stillfox
docker compose pull
for nom in serveur site; do
  attendu="$(awk -v p="ghcr.io/ppollet73/stillfox-$nom:" 'index($2, p) == 1 { print $1 }' empreintes.sha256)"
  image="$(grep -o "ghcr.io/ppollet73/stillfox-$nom@sha256:[0-9a-f]*" docker-compose.yml)"
  if docker image inspect --format '{{range .RepoDigests}}{{println .}}{{end}}' "$image" \
    | grep -qx "ghcr.io/ppollet73/stillfox-$nom@sha256:$attendu"; then
    echo "$nom : empreinte conforme"
  else
    echo "$nom : EMPREINTE DIFFÉRENTE de empreintes.sha256, ne démarrez pas"
  fi
done
docker compose up -d     # démarre aussi Caddy (section 6)
docker compose ps        # serveur et site doivent devenir « healthy » (une minute environ), caddy « running »
```

1. **Le code d'installation.** Tant qu'aucun compte n'existe, le serveur écrit dans son journal un code à usage unique, valable
   jusqu'à son prochain redémarrage. Lisez-le :

```sh
docker compose logs serveur | grep 'installation code'
```

2. Ouvrez `https://exemple.fr` : l'écran d'installation demande le code, vos identifiants (adresse e-mail, mot de passe de
   12 caractères au moins) et l'adresse publique du site (préremplie avec celle du navigateur ; `https` obligatoire, sauf
   sur `localhost`). Vous devenez titulaire du foyer et administrateur. Après le premier compte, l'écran se ferme : le code ne
   sert plus.
3. Puis le site propose de **créer une nouvelle centrale** ou d'**importer vos données depuis myfox.me** (réglage « Application
   Somfy » : vos propres identifiants de client de l'API Myfox, ceux du prérequis). L'import lit votre compte myfox.me en
   lecture seule (appareils, scénarios, contacts, historique, factures) ; chaque passage s'ajoute au précédent, sans rien
   remplacer. Dernier passage possible : avant le 31 mars 2027. « Mon compte > Paramètres avancés > Importer depuis myfox.me »
   le relance plus tard.

Pour l'envoi des e-mails (inscription, mot de passe oublié, alertes) et des notifications mobiles, ouvrez l'onglet
« Notifications » de l'Administration : relais SMTP, jeton Pushover, puis un envoi d'essai. Tant qu'aucun relais SMTP n'est
réglé, l'inscription et « Mot de passe oublié » restent fermés.

Sans navigateur, on peut contrôler depuis le serveur : `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8480/v2/setup`
répond 200 (l'état de l'installation) et `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8480/v2/openapi.yaml`
répond 401 sans jeton.

### Annexe : l'import en ligne de commande

Pour importer sans passer par le site, **serveur arrêté** (`docker compose stop serveur`), avec l'outil `myfox-api` de l'image
du serveur. Les identifiants myfox.me sont saisis au clavier ; l'extraction a aussi besoin de **vos** identifiants de client de
l'API Myfox, passés sans valeur sur la ligne de commande. Ne les mettez pas dans `.env` :

```sh
cd /opt/stillfox
mfx() { docker compose run --rm --no-deps -e MYFOX_CLIENT_ID -e MYFOX_CLIENT_SECRET \
    --entrypoint myfox-api serveur --db /data/hc2.sqlite "$@"; }

read -r -p "Identifiant du client Somfy : " MYFOX_CLIENT_ID
read -r -s -p "Secret du client Somfy : " MYFOX_CLIENT_SECRET; echo
export MYFOX_CLIENT_ID MYFOX_CLIENT_SECRET

mfx extract --out /data/paquets/premier
mfx import --household principal --bundle /data/paquets/premier
unset MYFOX_CLIENT_ID MYFOX_CLIENT_SECRET
```

Le paquet reste dans `data/paquets` : c'est aussi une sauvegarde lisible de vos données. La fonction `mfx` sert encore à la
restauration (section 10) et au rattachement de la centrale (section 8).

## 8. Rattacher votre centrale et la rediriger vers votre serveur

### Rattacher la centrale à votre foyer

La passerelle n'accepte une centrale que depuis l'adresse publique de son domicile **et** avec son adresse MAC. Depuis le
site, « Ma centrale » propose une demande de rattachement (adresse MAC, adresse publique) que l'administrateur approuve ;
avec un seul administrateur, elle est approuvée d'office. La passerelle relit la liste des foyers toutes les 30 secondes et dès que l'administrateur clique sur « Recharger la passerelle » (page d'administration, « Pare-feu des centrales ») : aucun redémarrage n'est nécessaire. Vous pouvez aussi la faire en ligne de commande (fonction `mfx` de
l'annexe de la section 7), serveur arrêté. Avec `--mode local`, la passerelle est lancée avec `--panels-local` et l'adresse publique est facultative dans « Ma centrale » : la centrale du réseau local est reconnue par son adresse MAC. Sinon, l'adresse publique doit être une vraie adresse publique : une adresse privée ou
réservée, comme celles des exemples de ce guide, est refusée (« not a public address ») :

```sh
docker compose stop serveur
mfx panel-attach --site <numéro du site> --mac AA:BB:CC:DD:EE:FF --ipv4 203.0.113.7
docker compose start serveur     # la passerelle relit la liste des foyers à son démarrage, puis toutes les 30 secondes
```

### Rediriger la centrale

Au démarrage, la centrale cherche les noms `core.steadyfox.com` et `hc2.steadyfox.com` dans le DNS que lui donne votre box.
Il faut que, **sur votre réseau local seulement**, ces deux noms répondent l'adresse de votre serveur (`192.0.2.10`) : une
règle de « réécriture » ou d'enregistrement local dans le serveur DNS de votre réseau (box, NAS, routeur, Pi-hole,
Unbound…). Laissez les autres noms inchangés. La centrale ouvre alors sa connexion sur le **port 8888** de votre serveur
(et un appel HTTP sur le port 80, auquel répond le bloc `hc2.steadyfox.com` de l'exemple Caddy).

**Mode local : ne redirigez jamais le port 8888 de votre box vers cette instance.** Une centrale rattachée sans adressepublique est admise depuis toute adresse privée : rediriger le port 8888 depuis Internet l'exposerait à n'importe quelappareil de ces réseaux. Le port 8888 ne sert qu'à la centrale, sur votre réseau local.
Contrôle : la dernière trame reçue s'affiche dans la page « Passerelle » du site, et l'état de la centrale y passe à
« connectée ».

### Pare-feu : le port 8888 seulement pour les foyers

Le port 8888 n'a pas d'authentification forte : ouvrez-le **uniquement** aux adresses publiques des foyers (ici
`203.0.113.7`). Docker contourne les règles ordinaires d'un pare-feu pour les ports qu'il publie : placez la règle avant
lui, par exemple avec nftables (table chargée au démarrage, avant Docker) :

```nft
table inet stillfox {
	chain avant {
		type filter hook prerouting priority -150; policy accept;
		tcp dport 8888 ip saddr != { 203.0.113.7 } drop
		tcp dport 8888 meta nfproto ipv6 drop
	}
}
```

Ajoutez une adresse entre les accolades à chaque nouveau foyer, puis relancez le serveur pour qu'il relise la liste. Vérifiez depuis
une machine extérieure que le port 8888 est refusé.

## 9. Mettre à jour vers une nouvelle version (à la main)

Le script le fait pour vous (`sudo bash stillfox.sh mettre-a-jour`, voir plus haut) ; voici la même chose à la main.

1. Lisez le journal des versions (`CHANGELOG.md`), et en particulier la section **« À faire avant la mise à jour »** d'une
   version à installer, s'il y en a une : faites-la d'abord.
2. Notez la version et l'empreinte (digest) en service pour pouvoir revenir en arrière :

```sh
cd /opt/stillfox
docker compose images
grep -o 'stillfox-[a-z]*@sha256:[0-9a-f]*' docker-compose.yml   # les empreintes en service
```

3. Copiez la base **serveur arrêté** (le fichier est verrouillé tant qu'il tourne) :

```sh
docker compose stop
cp -a data data.avant-0.4.1
```

4. Téléchargez `docker-compose.yml` et `empreintes.sha256` de la nouvelle release, contrôlez-les (même commande qu'à la section 4).
   Gardez l'ancien fichier compose (`mv docker-compose.yml docker-compose.yml.avant-0.4.1`) avant de mettre le nouveau à sa place :
   c'est lui qui désigne les images de la nouvelle version par leur empreinte.
5. Tirez les images, faites le même contrôle des empreintes qu'à la section 7 (la boucle `for nom in serveur site`, avec le
   nouveau `empreintes.sha256`), puis démarrez :

```sh
docker compose pull
# … la boucle de contrôle des empreintes de la section 7 …
docker compose up -d
docker compose ps        # « healthy » attendu ; la centrale se reconnecte seule en une minute environ
```

**Retour arrière** : `docker compose down`, remettre l'ancien `docker-compose.yml` (`docker-compose.yml.avant-0.4.1`, qui désigne les anciennes images par leur empreinte),
rendre sa base à l'ancienne version (`rm -rf data && cp -a data.avant-0.4.1 data`), puis `docker compose up -d`. Une base
copiée avant une mise à jour qui a changé son schéma est la seule façon sûre de revenir : ne la supprimez qu'après
quelques jours d'observation.

**Une installation faite avant la version 0.7.0** (avec un `.env`, une clé des réglages dans `secrets/` et deux identifiants
`MYFOX_API_CLIENT_*`) ne se met pas à jour en remplaçant simplement le compose : le nouveau compose lit d'autres fichiers et ignore
ces variables. Gardez votre compose tant que ce passage n'est pas décrit dans les notes de la version concernée.

## 10. Sauvegardes

- Le serveur écrit **chaque jour une sauvegarde chiffrée** dans `data/sauvegardes` (clé : `cle-sauvegardes`).
- Les paquets d'import (`data/paquets`) sont des copies lisibles de vos données myfox.me.
- Copiez régulièrement `/opt/stillfox/data` **hors du serveur** avec l'outil de votre choix (restic, borg, rsync…), ainsi que
  `secrets/` (les clés générées par le serveur y sont : sans elles, les réglages enregistrés ne s'ouvrent plus), que vous gardez
  à part des données : une sauvegarde sans sa clé est inutilisable, les clés ne vivent pas dans la même sauvegarde.
- Restaurer un foyer, serveur arrêté : `mfx household-restore --backup /data/sauvegardes/<numéro du foyer>/<horodatage>.sqlite.enc` (chemin vu depuis le conteneur). La restauration ajoute le foyer et ne remplace rien : sur une base qui contient déjà ce foyer, elle échoue (« constraint failed »). Elle sert à reconstruire une base neuve ou à rouvrir un foyer supprimé.
- Essayez une restauration une fois, avant d'en avoir besoin.

## Dépannage

| Symptôme | Piste |
|---|---|
| Le code d'installation est introuvable | `docker compose logs serveur \| grep 'installation code'` ; il est écrit à chaque démarrage tant qu'aucun compte n'existe ; il change à chaque redémarrage |
| L'écran d'installation refuse le code | code d'un démarrage précédent, ou trop d'essais : redémarrez le serveur (`docker compose restart serveur`) et relisez le code |
| Le serveur ne démarre pas, erreur d'écriture | `data` ou `secrets/generes` n'appartient pas à l'uid 10001 ou n'est pas en 700 |
| Erreur sur la clé de sauvegarde | fichier absent, pas en 400, ou propriétaire différent de 10001 |
| Le site ne démarre pas | il attend la santé du serveur : `docker compose logs serveur` |
| La centrale ne se connecte pas | rattachement (adresse MAC, adresse publique), redirection DNS, pare-feu 8888, attendre 30 secondes ou « Recharger la passerelle » après un changement de foyer |
| Pas de certificat, le navigateur refuse `https://exemple.fr` | `docker compose logs caddy` : nom qui ne pointe pas encore vers le serveur, port 80 ou 443 fermé ou déjà pris par un autre serveur web |
| Connexion refusée, « Host » ou « Origin » | l'adresse publique enregistrée à l'installation ne correspond pas à celle du navigateur (Administration) |

---

stillfox est un projet indépendant, ni affilié ni approuvé par Somfy. Démonstration publique : <https://ppollet73.github.io/stillfox/>.
