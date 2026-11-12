# Journal des versions

Ce journal ne liste que les nouveautés et corrections visibles par un foyer ou un administrateur de stillfox.
Format : [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/) ; numérotation : [SemVer](https://semver.org/lang/fr/).

## [Non publié]

## [0.5.1] - 2026-11-12

### Ajouté

- Applications personnelles : chaque compte peut créer sa propre application pour ses scripts, avec un identifiant et un secret (montré une seule fois, renouvelable). L'administrateur peut suspendre ou réactiver une application.
- Portail développeur sur le site (`/developpeur`) : introduction, authentification, documentation interactive de l'API, gestion de son application et conditions d'utilisation.
- Administration : onglet « Applications personnelles » et onglet « Interrupteurs » avec un huitième interrupteur, « Nouveaux appareils », qui autorise l'apprentissage de badges, de télécommandes et de touches tactiles (coupé par défaut).
- Journal des trames : une bulle « En clair » traduit chaque trame, avec le nom de l'appareil concerné.
- Contacts d'alerte : fiche de chaque contact avec toutes ses alertes à cocher, ajout et retrait de contacts (4 au plus). Le titulaire reste le contact par défaut et le dernier contact ne peut pas être retiré.
- Perte de l'alimentation secteur de la centrale, rétablissement et redémarrage : alerte et ligne d'historique (détection encore à confirmer sur une installation réelle).
- Une ligne d'historique signale un appareil qui n'est plus à portée radio de la centrale.
- Icônes plus explicites : libellés visibles dans l'en-tête, pastilles « Internet », « Secteur » et « Diagnostic » de la centrale.
- Retour plus rapide vers Somfy après une panne : nouvel essai dès 1 minute.
- Mise à l'heure de la centrale suspendue quand l'horloge de la machine paraît fausse.

### Corrigé

- « Gérer mes contacts » s'affiche de nouveau.
- Un compte relié à Google garde son mot de passe pour ses applications personnelles ; un rappel figure sur la page de connexion.
- Le bouton « Continuer avec Google » ouvre bien la connexion et a la taille du bouton « Connexion ».
- Les applications autorisées de « Mon compte » ont un nom.
- Installation : l'adresse publique mal formée ou en `http` est refusée, et l'inscription par e-mail ou Google n'est plus proposée tant que l'installation est ouverte.
- Apprentissage de badges, de télécommandes et d'IntelliTAG : il est refusé tant que « Nouveaux appareils » est coupé, et une séance est toujours refermée.
- La courbe d'un appareil ouvre le même graphique que l'accueil ; « Reconnaître mes appareils » montre la vignette de chaque appareil.
- La règle « 12 caractères au moins » du mot de passe passe au vert quand elle est remplie ; les interrupteurs des réglages d'appareil et le bouton du relais se lisent plus facilement.
- Libellé « Perte signal radio » au lieu de « Perdu par radio ».
- L'ancien portail développeur du serveur renvoie vers le nouveau.

## [0.5.0] - 2026-11-11

### Ajouté

- Installation sur un serveur vide : un écran d'installation crée le premier compte administrateur et demande l'adresse publique du site ; plus de fichier de réglages à écrire à la main, les clés secrètes sont générées par le serveur.
- Import depuis myfox.me : l'administrateur saisit d'abord l'identifiant et le secret de l'application Somfy, puis l'import continue.
- Chauffage et volets roulants : commandes de mode (jour, nuit, arrêt, hors gel) et d'ouverture, fermeture ou position favorite, à activer par l'administrateur.
- Administration : interrupteurs pour autoriser ou couper les commandes réelles, les appareils DIO, les scénarios et la mise à l'heure des centrales (tout est coupé après la mise à jour).
- Administration : onglet « Journal des envois » listant chaque e-mail, notification mobile et alerte HTTPS envoyé, avec son résultat, sur 90 jours ; les destinataires y sont masqués.
- Administration : bouton « Envoyer un essai » pour les comptes d'envoi e-mail et Pushover, et aide pour créer l'application Pushover.
- Adresse du site facultative, modifiable aussi dans « Mon compte ».
- Le mode d'alerte réglé sur un appareil est respecté : un canal désactivé n'envoie plus l'alarme de cet appareil.
- Sauvegarde nocturne chiffrée de l'ensemble du serveur, avec une commande de restauration.
- Scénarios : proposition d'emplacements à confirmer pour les scénarios d'un nouveau foyer.
- Nouvel habillage sur tous les écrans, en clair et en sombre : accueil avec bandeau « Ma protection », registre des appareils avec recherche, scénarios, alertes, historique, administration, aide et médiathèque.
- Fenêtres accessibles au clavier (Échap, focus conservé) et site qui ne charge plus aucun fichier venant de myfox.me.

### Corrigé

- Un scénario écrit par myfox.me avec des actions de volets n'est plus effacé en silence quand on le modifie ou le réactive : il est refusé avec un message clair.
- Plusieurs foyers : les alarmes, l'historique et les ordres de chaque foyer restent chez ce foyer, et un serveur ambigu prévient les administrateurs au lieu de se tromper de foyer.
- Import : un travail d'import s'annule si le titulaire change ou si son compte n'est plus actif.
- La page d'accueil du portail développeur répond au lieu d'une erreur 404.
- Le titre de l'en-tête n'est plus coupé sur un écran de taille moyenne, et une session expirée ne renvoie plus qu'une seule fois vers la connexion.
- Les réglages de la centrale s'ouvrent avec son identifiant propre, comme sur myfox.me.

## [0.4.1] - 2026-11-10

### Ajouté

- Mot de passe oublié : lien de réinitialisation par e-mail, valable 30 minutes et utilisable une seule fois.
- Alertes envoyées pour de bon par e-mail et par notification mobile (Pushover).
- Chaque contact d'alerte renseigne sa propre clé Pushover dans « Gérer mes contacts », avec une aide pour la trouver.
- « Mes alertes » : case pour être prévenu des changements de niveau de protection faits depuis le site ou par un scénario (désactivée par défaut).
- Administration : onglet « Notifications » pour régler les comptes d'envoi (e-mail, Pushover).
- Administration : onglet « Notifications des foyers » pour régler les contacts et les niveaux d'un foyer.
- Alertes transmises aussi à une adresse HTTPS par foyer, par exemple pour Home Assistant, avec un envoi d'essai.
- Page « Passerelle » : l'état de la centrale s'affiche (connectée par le relais, connectée directement, ou déconnectée).
- Import depuis myfox.me : les appareils des groupes sont repris avec leurs groupes.
- Appairage d'un IntelliTAG : le support (fenêtre, porte coulissante…) est mémorisé.
- Nouvel habillage : barre latérale, niveau de protection toujours visible en haut de page, thème clair ou sombre, menu repliable sur petit écran.

### Corrigé

- Un IntelliTAG en détection par vibration n'affiche plus de demande de recalibrage, comme sur myfox.me.
- Plusieurs foyers sur un même serveur : les ordres de chaque foyer ne vont plus qu'à sa propre centrale.

## [0.4.0] - 2026-11-09

### Ajouté

- Scénarios : créer, modifier, activer, désactiver, lancer et supprimer un scénario depuis le site, avec un assistant en quelques étapes.
- Créer un site sans import, puis y rattacher sa centrale en indiquant son adresse MAC ; l'administrateur approuve la demande.
- Réglages d'un appareil : « Paramétrer mon appareil » enregistre les changements (nom, sensibilité, décalage de température, niveaux de détection).
- Un réglage dont la valeur est inconnue s'affiche « inconnu » au lieu d'une valeur fausse.
- Alerte « centrale hors ligne » et rappel de pile faible, envoyés aux contacts du foyer.
- La passerelle répond elle-même à la centrale quand le service de Somfy ne répond plus.
- Groupes d'appareils : créer un groupe, régler ses membres, allumer ou éteindre un groupe électrique.
- Autoprotection : confirmation pour la sirène extérieure.
- Catalogue « Ajouter un appareil » : toutes les familles affichent toutes leurs tuiles ; attente du geste de calibrage allongée à 60 secondes.
- « Ajouter un site » s'affiche en pleine page.
- Administration : onglet « Pare-feu des centrales » pour autoriser l'adresse d'une nouvelle centrale.
- « Reconnaître mes appareils » plus fiable : un appareil n'est proposé qu'après deux écoutes concordantes, et l'assistant dit quand un seul appareil a parlé.
- L'historique reprend le nom de l'appareil dès qu'il est relié à son numéro dans la centrale.
- Volets RTS et modules de chauffage reconnus par la passerelle.
- Limite de tentatives de connexion par adresse.
- Les journaux d'un foyer supprimé sont effacés au bout de 30 jours.
- Pied de page : année en cours et version réelle du site.

### Corrigé

- Un module de chauffage dont l'accusé de réception était pris à tort pour un refus est désormais bien reconnu.
- La page de connexion ne se recharge plus en boucle quand la session a expiré.

## [0.3.0] - 2026-11-06

### Ajouté

- « Continuer avec Google » à la connexion et à l'inscription.
- Changer son mot de passe ; révoquer l'accès d'une application.
- Commandes réelles vers la centrale : armement, désarmement, prises, appareils.
- Ajouter un appareil (télécommande, IntelliTAG), le retirer, le recalibrer, le supprimer depuis le site.
- Désactiver l'autoprotection de la sirène extérieure ; groupes d'appareils.
- Icônes Internet et alimentation de l'en-tête alimentées par l'état réel de la centrale.
- Diagnostic : « connexion perdue » en tête quand la centrale ne répond plus depuis plus de deux minutes.
- Historique : défaut de la centrale, installation d'un appareil, recalibration et suppression apparaissent dans le journal.
- Plusieurs foyers sur un même serveur, chacun reconnu à l'adresse publique et à l'adresse MAC de sa centrale.
- Import de myfox.me répétable jusqu'au 31 mars 2027, avec un rapport des écarts.
- Détection d'une centrale coupée en 40 secondes au plus.
- Plus aucune donnée n'est demandée à myfox.me en dehors de l'import.

### Corrigé

- Plus de commande simulée présentée comme réelle : une commande sans effet réel est refusée au lieu de répondre « OK ».

## [0.2.0] - 2026-11-05

### Ajouté

- Inscription en libre-service par e-mail, avec validation par l'administrateur.
- Administration des comptes.
- Rapport d'import lisible : ce qui a été lu et ce qui était déjà présent.
- « Mes alertes » relié aux vraies règles : niveaux de protection, contacts, journal des alertes.
- Journal des trames : recherche, pagination (25 à 200 trames par page), colonnes redimensionnables.
- Page « Passerelle » dans « Mon système » ; le relais peut être coupé et rétabli depuis le site.
- Le tableau de bord suit l'alarme en direct (armement à la télécommande pris en compte sans recharger).
- Alertes « défaut d'accessoire » (perte radio) et « pile faible ».
- Adresse radio des appareils dans la liste de la centrale.
- Lien vers le portail développeur de l'API.

### Corrigé

- Une pile ou une perte radio mal lue dans le diagnostic ne s'affiche plus à tort.
- Un diagnostic ancien n'écrase plus un état plus récent.
- Un mot de passe d'inscription est limité à 256 caractères, comme sur le site.
- « Reconnaître mes appareils » : l'écoute se termine dès qu'un appareil candidat a parlé.

## [0.1.0] - 2026-11-04

### Ajouté

- Première version à installer : relais vers Somfy en mode « ombre », base et site hébergés sur son propre serveur.
- Import des foyers depuis myfox.me : écran « Importer depuis myfox.me », paquet téléchargeable.
- « Reconnaître mes appareils » : assistant pour retrouver le numéro de chaque appareil dans la centrale.
- Page « Passerelle » : mode du relais, dernière trame reçue.
- Alertes par e-mail et notification mobile.
- Sauvegardes quotidiennes chiffrées, une par foyer.
- Versions du site et de l'API dans le pied de page.

## [0.0.9] - 2026-11-02

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Plusieurs foyers sur un même serveur, avec comptes, mots de passe protégés et droits par site.
- Les événements de la centrale alimentent le niveau de protection, le diagnostic, la température et l'historique.
- Contacts, alertes, notifications et noms d'appareil modifiables.
- Scénarios et factures de myfox.me repris à l'import.

### Corrigé

- Compte verrouillé ou service indisponible : le site l'indique au lieu de fermer une session valide.

## [0.0.8] - 2026-11-01

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Premières écritures (niveau de protection, scénarios) essayées sur une base d'essai.

## [0.0.7] - 2026-11-01

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Serveur local qui rejoue l'API de myfox.me : les 38 appareils du foyer, dont les capteurs TAG absents de l'API officielle.

### Corrigé

- Un scénario à la demande sans état d'activation ne fait plus échouer la liste des scénarios.

## [0.0.6] - 2026-11-01

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Le site affiche les données que l'API officielle ne donne pas (nom du titulaire, crédits, diagnostic, contacts, réglages, factures).

### Corrigé

- Le contenu des scénarios est lu comme sur myfox.me.

## [0.0.5] - 2026-10-15

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Page de connexion, « Mon compte » (identité, facturation, factures), « Mes alertes » et « Aide » avec ses notes de version.
- « Paramétrer mon appareil » et écrans de scénarios, en lecture seule.
- « Mon historique » : dates choisies au sélecteur, affichage dès la première tranche lue.

### Corrigé

- Graphique de température : l'infobulle n'est plus vide sur certains relevés.

## [0.0.4] - 2026-10-14

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- « Mon système », « Mon historique » sur un an, « Mes scénarios », « Mes sites », « Mes graphiques » de température.

### Corrigé

- Les interrupteurs de scénario du tableau de bord répondent de nouveau au clic.

## [0.0.3] - 2026-10-14

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Tableau de bord au gabarit de myfox.me : protection, scénarios, prises, données sans API signalées par un badge.

## [0.0.2] - 2026-10-14

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Première lecture réelle du compte (connexion, listes, températures, historique), en lecture seule.

## [0.0.1] - 2026-10-13

_Approximatif : cette section a été reconstituée après coup et peut être incomplète._

### Ajouté

- Première version : fondations de la reconstruction de l'interface myfox.me.
