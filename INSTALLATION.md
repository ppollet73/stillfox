# Installer stillfox sur son propre serveur

stillfox se compose de deux conteneurs : le **serveur** (la base de données, l'API et la passerelle qui parle à votre
centrale HC2 sur le port 8888) et le **site** (l'interface web). Ce guide les installe à partir des images publiques et
des fichiers joints à chaque release. Les exemples utilisent `exemple.fr` pour le nom de domaine, `192.0.2.10` pour
l'adresse de votre serveur et `203.0.113.7` pour l'adresse publique de votre domicile : remplacez-les par les vôtres.

Les images publiques :

- `ghcr.io/ppollet73/stillfox-serveur:<version>`
- `ghcr.io/ppollet73/stillfox-site:<version>`

Chaque version est aussi désignée par l'empreinte sha256 (digest) de ses images : le `docker-compose.yml` joint à la
release écrit `image: ghcr.io/ppollet73/stillfox-serveur@sha256:…` (et de même pour le site). Il ne peut donc tirer que les
images publiées avec cette release, même si une étiquette de version était déplacée plus tard. Il n'y a pas de numéro de
version à renseigner : changer de version, c'est remplacer ce fichier (section 9).

Chaque release joint trois fichiers : `docker-compose.yml`, `.env.example` et `empreintes.sha256`. Le
`docker-compose.yml` est générique : il ne construit rien (il tire les images publiques), ne fixe ni sous-réseau ni
adresse, et refuse de démarrer tant que les variables obligatoires de votre `.env` manquent.

## 1. Prérequis

- Un serveur Linux (un petit VPS suffit : le serveur est borné à 384 Mo de mémoire et le site à 192 Mo), avec **une adresse
  IPv4 publique fixe**.
- Docker et Docker Compose (le plugin `docker compose`).
- Un nom de domaine, avec deux noms qui pointent vers le serveur : `exemple.fr` (le site) et `api.exemple.fr` (l'API).
- Les ports **80 et 443** du serveur ouverts sur Internet et libres (aucun autre serveur web) : **Caddy**, installé à la
  section 6 dans le même compose, s'y place devant le site et l'API et obtient seul les certificats HTTPS. Le site
  refuse de fonctionner sans HTTPS : le cookie de session est « Secure ». Si votre serveur a déjà un proxy inverse,
  gardez-le et reprenez seulement le contenu du `Caddyfile` de la section 6.
- Un compte myfox.me qui fonctionne encore : l'import de vos données s'y connecte. **Après le 31 mars 2027, il ne sera plus
  possible de les récupérer.**
- **Vos propres identifiants de client de l'API Myfox** (`MYFOX_CLIENT_ID` et `MYFOX_CLIENT_SECRET`), obtenus sur le
  portail développeur de myfox.me, tant qu'il existe. Chaque installateur utilise les siens : ceux du projet ne sont jamais
  publiés. Ils ne servent qu'à l'import initial (étape 7) ; vous ne les écrivez pas dans `.env`.
- Facultatif : un relais SMTP (inscription et mot de passe oublié par e-mail), un compte Pushover (notifications mobiles).

## 2. Arborescence

Tout vit dans `/opt/stillfox`. Créez-la en root :

```sh
install -d -m 755 /opt/stillfox
install -d -o 10001 -g 10001 -m 700 /opt/stillfox/data
install -d -m 700 /opt/stillfox/secrets
install -d -m 700 /opt/stillfox/caddy /opt/stillfox/caddy/data /opt/stillfox/caddy/config
```

```
/opt/stillfox/
├── data/                       uid 10001, droits 700 : base, journaux, sauvegardes, paquets d'import
├── secrets/                    root, droits 700
│   ├── cle-sauvegardes         uid 10001, droits 400
│   └── cle-reglages            uid 10001, droits 400
├── caddy/                      root, droits 700 : certificats HTTPS (data) et état de Caddy (config)
├── Caddyfile                   root, droits 644 (section 6)
├── docker-compose.yml          root, droits 644 (fichier joint à la release)
├── docker-compose.override.yml root, droits 644 : le service Caddy (section 6)
└── .env                        root, droits 600
```

Le serveur tourne sous l'utilisateur **10001** dans le conteneur : un dossier de données qui appartient à root, ou une clé
en 600 appartenant à root, lui seraient illisibles. Le dossier `data` doit exister **avant** le premier démarrage ;
Docker ne le crée pas.

## 3. Les deux clés de fichier

Le serveur chiffre ses sauvegardes avec la première clé, et les secrets qu'il garde en base (mot de passe SMTP, jeton
Pushover…) avec la seconde. Chacune fait 64 caractères hexadécimaux :

```sh
openssl rand -hex 32 > /opt/stillfox/secrets/cle-sauvegardes
openssl rand -hex 32 > /opt/stillfox/secrets/cle-reglages
chown 10001:10001 /opt/stillfox/secrets/cle-*
chmod 400 /opt/stillfox/secrets/cle-*
```

**Gardez une copie de ces deux fichiers hors du serveur** (gestionnaire de mots de passe, autre machine). Sans la clé des
sauvegardes, vos sauvegardes sont illisibles ; sans la clé des réglages, les secrets enregistrés ne peuvent plus être ouverts.

## 4. Télécharger les fichiers et contrôler les empreintes

Depuis la page de la release choisie, téléchargez les trois fichiers dans `/opt/stillfox` (adaptez l'adresse à la
version voulue). Puis contrôlez-les :

```sh
cd /opt/stillfox
grep -E ' (docker-compose.yml|[.]env.example)$' empreintes.sha256 | sha256sum -c
```

Chacune des deux lignes doit afficher `OK`. Une ligne en échec : ne continuez pas, retéléchargez le fichier. (Les
lignes suivantes de `empreintes.sha256` donnent les empreintes des images ; ce ne sont pas des fichiers, d'où le
`grep`.)

```sh
mv .env.example .env
chmod 600 .env
```

## 5. Le fichier `.env`

Ouvrez `.env` et remplissez-le en suivant les commentaires du fichier. Dans le fichier joint,
`HC2_PUBLIC_ORIGIN`, `HC2_PUBLIC_API_URL` et `MYFOX_SITE_URL` sont **commentées** (précédées de `#`) : retirez le `#` et
donnez-leur votre valeur (dans `.env.example`, `<votre domaine>` est un marqueur à remplacer). Toutes les variables du
tableau sont **obligatoires**, sauf `MYFOX_BUNDLE_KEY` : sans l'une d'elles, `docker compose` s'arrête avec un message
du type `set VPS_IPV4 in .env (…)`. Les valeurs à fournir :

| Variable | Rôle |
|---|---|
| `VPS_IPV4` | adresse IPv4 publique du serveur (`192.0.2.10` dans nos exemples) : le port 8888 est publié sur cette adresse seulement |
| `MYFOX_API_CLIENT_ID` | identifiant que le site présente à l'API de votre serveur, **à inventer** : `openssl rand -hex 16`. Sans rapport avec les identifiants Somfy de l'import |
| `MYFOX_API_CLIENT_SECRET` | secret de cette application : `openssl rand -hex 32` |
| `MYFOX_BUNDLE_KEY` | clé de signature des paquets d'import : `openssl rand -hex 32` |
| `HC2_PUBLIC_ORIGIN` | adresse publique du site, par exemple `https://exemple.fr` |
| `HC2_PUBLIC_API_URL` | adresse publique de l'API, par exemple `https://api.exemple.fr` |
| `MYFOX_SITE_URL` | adresse publique du site, utilisée dans les liens des e-mails (HTTPS obligatoire) |

Le secret de l'application cliente et la clé des paquets peuvent aussi se donner **par fichier** (secret Docker, comme les
autres clés), au lieu de passer par `.env` et donc par `docker inspect` : `MYFOX_API_CLIENT_SECRET_FILE` et
`MYFOX_BUNDLE_KEY_FILE` (et `MYFOX_CLIENT_SECRET_FILE` pour le site) nomment un fichier d'une seule ligne, lisible par le
conteneur. La variable seule reste acceptée ; **les deux à la fois** (variable non vide et fichier) : le serveur refuse de
démarrer, et un fichier illisible ou vide aussi.

Une ligne `HC2_VERSION` peut subsister dans `.env.example` (elle sert au compose de notre propre serveur) : le compose
joint à la release ne la lit pas, ignorez-la.

Le fichier doit être en UTF-8 **sans BOM** et avec des fins de ligne **LF**. Un fichier écrit depuis Windows commence
par un BOM et Compose lit alors une variable « vide ». Pour réparer sur le serveur :

```sh
sed -i '1s/^\xEF\xBB\xBF//; s/\r$//' /opt/stillfox/.env
```

Ne publiez jamais ce fichier et ne l'affichez pas dans un journal.

## 6. Caddy, le proxy inverse HTTPS

Le site (port 8383) et l'API (port 8480) ne sont jamais ouverts à Internet : le `docker-compose.yml` joint les publie
**sur `127.0.0.1` seulement**. C'est Caddy qui reçoit les visiteurs sur les ports 80 et 443, obtient et renouvelle seul
les certificats HTTPS (Let's Encrypt), puis transmet au site et à l'API. Il tourne dans un conteneur en réseau « host »
pour les atteindre à `127.0.0.1:8383` et `127.0.0.1:8480` : ni sous-réseau ni adresse fixe à prévoir.

Avant de le démarrer, les deux noms `exemple.fr` et `api.exemple.fr` doivent déjà pointer vers le serveur et les ports 80
et 443 être ouverts : sinon Let's Encrypt ne peut pas valider le domaine et Caddy reste sans certificat (il réessaie seul).

### Le service Caddy

Créez `/opt/stillfox/docker-compose.override.yml`. Docker Compose le lit automatiquement avec `docker-compose.yml` :
`docker compose up -d` démarre alors les trois conteneurs, et la mise à jour de la section 9, qui ne remplace que
`docker-compose.yml`, ne le touche pas.

```yaml
services:
  caddy:
    image: caddy:2.11.9
    restart: unless-stopped
    network_mode: host          # écoute sur 80 et 443, joint le site et l'API sur 127.0.0.1
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

Validez le fichier, puis, Caddy une fois démarré (étape 7.3), rechargez-le après chaque modification, sans coupure :

```sh
cd /opt/stillfox
docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Caddy ne démarre qu'avec les deux autres conteneurs (`docker compose up -d`, étape 7.3). Contrôle :
`docker compose logs caddy | grep -i certificate` doit montrer « certificate obtained successfully » pour chacun des
deux noms, puis `https://exemple.fr` s'ouvre avec un cadenas.

### Adresse réelle du visiteur (facultatif)

Seul un pair déclaré a le droit d'annoncer l'adresse de son visiteur (adresses seules, pas de réseau), et par défaut
personne ne l'est : `HC2_TRUSTED_PROXIES` (le proxy, pour le site) et `MYFOX_API_TRUSTED_PEERS` (le conteneur du site,
pour l'API) sont vides. Conséquences pour une installation à un seul foyer, acceptables : tous les visiteurs apparaissent
avec la même adresse (les compteurs d'échecs de connexion et d'inscription sont partagés) et l'import lancé depuis le
site (« Importer depuis myfox.me ») est refusé ; l'import en ligne de commande de l'étape 7 n'est pas concerné.
Attention : sans pair de confiance, tous les visiteurs ont la même adresse aux yeux du serveur, donc un seul visiteur qui
multiplie les échecs de connexion peut bloquer les connexions de tous. Tant que vous n'avez pas déclaré de pair de confiance,
faites le filtrage par adresse dans le proxy (n'admettre que les adresses qui doivent accéder au site, voir plus haut).


Pour lever ces limites, il faut des adresses connues : donnez au réseau du compose un sous-réseau et au site une
adresse fixe dans un fichier `docker-compose.override.yml` de votre choix (adresses d'exemple dans une plage réservée à la documentation : remplacez-les par un sous-réseau privé libre de votre machine), placez le proxy
sur ce réseau avec sa propre adresse fixe, puis renseignez les deux variables dans `.env`.

```yaml
services:
  site:
    networks:
      default:
        ipv4_address: 198.51.100.3
networks:
  default:
    ipam:
      config:
        - subnet: 198.51.100.0/24
```

Avec le proxy à `198.51.100.10` : `HC2_TRUSTED_PROXIES=198.51.100.10` et `MYFOX_API_TRUSTED_PEERS=198.51.100.3,198.51.100.10`
(la seconde adresse seulement si le bloc de l'API de votre proxy envoie lui-même l'adresse du visiteur, comme
l'exemple ci-dessus). Dans ce même fichier, le service `caddy` remplace alors `network_mode: host` par
`ports: ["80:80", "443:443"]` et une adresse fixe sur le réseau `default` (`ipv4_address: 198.51.100.10`), et le
`Caddyfile` joint `site:8383` et `serveur:8480` par leur nom au lieu de `127.0.0.1`. Option avancée : le reste du guide
ne la suppose pas.

## 7. Premier démarrage et import de vos données

**Une base vide ne démarre pas.** Au démarrage, le serveur exige qu'au moins un site ait été importé. Sinon son conteneur
reste « unhealthy » (le journal dit « API not started … nothing imported yet »), et le site, qui attend sa santé, ne démarre
pas. Il n'existe pas d'installation « à blanc » : le premier site vient toujours d'un import depuis un compte myfox.me, donc
**cette étape exige un vrai compte myfox.me**. On importe **avant** de démarrer. L'outil `myfox-api` est dans l'image du serveur : on le lance avec
`docker compose run`, serveur arrêté.

1. Extrayez puis importez. Les
   identifiants myfox.me sont saisis au clavier (jamais sur la ligne de commande) ; l'extraction a aussi besoin de
   **vos** identifiants de client de l'API Myfox, ceux du prérequis (variables `MYFOX_CLIENT_ID` et
   `MYFOX_CLIENT_SECRET`, passées sans valeur sur la ligne de commande). Ils ne servent qu'à cet import : ne les
   mettez pas dans `.env`, ne les publiez pas, ne les partagez pas. Ne les confondez pas avec la paire
   `MYFOX_API_CLIENT_ID` / `MYFOX_API_CLIENT_SECRET` de l'étape 5, que vous avez inventée :

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
ls data/paquets/premier                         # le dossier « site-<numéro> » donne le numéro de votre site
```

   `extract` lit votre compte myfox.me en lecture seule (appareils, scénarios, contacts, historique, factures) et écrit un
   paquet dans `data/paquets` ; il affiche seulement des nombres (`"sites": 1`…). `import` le range dans la base et
   répond `site: 1`. Le paquet est aussi une sauvegarde lisible de vos données : gardez-le. Notez le numéro de votre
   site (dossier `site-<numéro>`) : il ne s'écrit nulle part ailleurs et reste chez vous ; l'inscription de la centrale
   (`panel-attach --site`) le demande.

2. Créez votre compte stillfox (le mot de passe se saisit au clavier : 12 caractères au moins), faites-en le titulaire du
   foyer, puis administrateur. Les deux commandes n'affichent rien quand elles réussissent :

```sh
mfx account-add --household principal --username <votre-adresse-e-mail> --all-sites --holder
mfx account-admin --email <votre-adresse-e-mail>
```

3. Tirez les deux images, contrôlez leur empreinte, puis démarrez. Le fichier compose joint désigne chaque image par son
   empreinte sha256 : Docker refuse déjà une image dont le contenu ne correspond pas. Le contrôle ci-dessous le vérifie
   en plus, contre `empreintes.sha256` (l'empreinte que Docker a retenue pour chaque image est la ligne `RepoDigests`) :

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

4. Ouvrez `https://exemple.fr`, connectez-vous, vérifiez vos appareils et votre historique. Sans navigateur, on peut
   contrôler depuis le serveur : `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8480/v2/signup` répond 200
   (l'API) et `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8480/v2/openapi.yaml` répond 401 sans jeton.
   Plus tard, « Mon compte > Paramètres avancés > Importer depuis myfox.me » relance l'import depuis le site (il demande
   alors le réglage « Application Somfy » de l'Administration : vos propres identifiants de client, ceux du prérequis) : chaque passage s'ajoute au précédent, sans rien remplacer.
   Dernier passage possible : avant le 31 mars 2027.

Pour l'envoi des e-mails (inscription, mot de passe oublié, alertes) et des notifications mobiles, ouvrez l'onglet
« Notifications » de l'Administration : relais SMTP, jeton Pushover, puis un envoi d'essai. Tant qu'aucun relais SMTP n'est
réglé, l'inscription et « Mot de passe oublié » restent fermés.

## 8. Rattacher votre centrale et la rediriger vers votre serveur

### Rattacher la centrale à votre foyer

La passerelle n'accepte une centrale que depuis l'adresse publique de son domicile **et** avec son adresse MAC. Depuis le
site, « Ma centrale » propose une demande de rattachement (adresse MAC, adresse publique) que l'administrateur approuve.
Vous pouvez aussi la faire en ligne de commande (fonction `mfx` de l'étape 7), serveur arrêté. L'adresse publique doit être une vraie adresse publique : une adresse privée ou réservée, comme celles des exemples de ce guide, est refusée (« not a public address ») :

```sh
docker compose stop serveur
mfx panel-attach --site <numéro du site> --mac AA:BB:CC:DD:EE:FF --ipv4 203.0.113.7
docker compose start serveur     # la passerelle relit la liste des foyers à son démarrage
```

### Rediriger la centrale

Au démarrage, la centrale cherche les noms `core.steadyfox.com` et `hc2.steadyfox.com` dans le DNS que lui donne votre box.
Il faut que, **sur votre réseau local seulement**, ces deux noms répondent l'adresse de votre serveur (`192.0.2.10`) : une
règle de « réécriture » ou d'enregistrement local dans le serveur DNS de votre réseau (box, NAS, routeur, Pi-hole,
Unbound…). Laissez les autres noms inchangés. La centrale ouvre alors sa connexion sur le **port 8888** de votre serveur
(et un appel HTTP sur le port 80, auquel répond le bloc `hc2.steadyfox.com` de l'exemple Caddy).

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

## 9. Mettre à jour vers une nouvelle version

1. Lisez le journal des versions (`CHANGELOG.md`).
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

4. Téléchargez les trois fichiers de la nouvelle release, contrôlez-les (même commande qu'à la section 4) et comparez le
   nouveau `.env.example` à votre `.env` : une variable nouvelle doit y être ajoutée. Gardez l'ancien fichier compose
   (`mv docker-compose.yml docker-compose.yml.avant-0.4.1`) avant de mettre le nouveau à sa place : c'est lui qui désigne
   les images de la nouvelle version par leur empreinte.
5. Tirez les images, faites le même contrôle des empreintes qu'à l'étape 7.3 (la boucle `for nom in serveur site`, avec le
   nouveau `empreintes.sha256`), puis démarrez :

```sh
docker compose pull
# … la boucle de contrôle des empreintes de l'étape 7.3 …
docker compose up -d
docker compose ps        # « healthy » attendu ; la centrale se reconnecte seule en une minute environ
```

**Retour arrière** : `docker compose down`, remettre l'ancien `docker-compose.yml` (`docker-compose.yml.avant-0.4.1`, qui désigne les anciennes images par leur empreinte),
rendre sa base à l'ancienne version (`rm -rf data && cp -a data.avant-0.4.1 data`), puis `docker compose up -d`. Une base
copiée avant une mise à jour qui a changé son schéma est la seule façon sûre de revenir : ne la supprimez qu'après
quelques jours d'observation.

## 10. Sauvegardes

- Le serveur écrit **chaque jour une sauvegarde chiffrée par foyer** dans `data/sauvegardes` (clé : `cle-sauvegardes`).
- Les paquets d'import (`data/paquets`) sont des copies lisibles de vos données myfox.me.
- Copiez régulièrement `/opt/stillfox/data` **hors du serveur** avec l'outil de votre choix (restic, borg, rsync…), en
  excluant `secrets/` et `.env`, que vous gardez à part. Une sauvegarde sans sa clé est inutilisable : les deux
  clés vivent ailleurs que dans la même sauvegarde.
- Restaurer un foyer, serveur arrêté : `mfx household-restore --backup /data/sauvegardes/<numéro du foyer>/<horodatage>.sqlite.enc` (chemin vu depuis le conteneur). La restauration ajoute le foyer et ne remplace rien : sur une base qui contient déjà ce foyer, elle échoue (« constraint failed »). Elle sert à reconstruire une base neuve ou à rouvrir un foyer supprimé.
- Essayez une restauration une fois, avant d'en avoir besoin.

## Dépannage

| Symptôme | Piste |
|---|---|
| Le serveur reste « unhealthy », journal « API not started … nothing imported yet » | rien n'a été importé : reprenez l'étape 7 |
| `required variable X is missing` ou `set X in .env (…)` | la variable `X` manque ou est vide dans `.env` (étape 5) |
| Le serveur ne démarre pas, erreur d'écriture | `data` n'appartient pas à l'uid 10001 ou n'est pas en 700 |
| Erreur sur la clé de sauvegarde ou de réglages | fichier absent, pas en 400, ou propriétaire différent de 10001 |
| `VPS_IPV4` « vide » alors que le fichier est rempli | BOM ou fins de ligne Windows dans `.env` (étape 5) |
| La centrale ne se connecte pas | rattachement (adresse MAC, adresse publique), redirection DNS, pare-feu 8888, redémarrage du serveur après un changement de foyer |
| Pas de certificat, le navigateur refuse `https://exemple.fr` | `docker compose logs caddy` : nom qui ne pointe pas encore vers le serveur, port 80 ou 443 fermé ou déjà pris par un autre serveur web |
| Connexion refusée, « Host » ou « Origin » | `HC2_PUBLIC_ORIGIN` ne correspond pas à l'adresse utilisée dans le navigateur |

---

stillfox est un projet indépendant, ni affilié ni approuvé par Somfy. Démonstration publique : <https://ppollet73.github.io/stillfox/>.
