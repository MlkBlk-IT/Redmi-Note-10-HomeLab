HOME-LAB ANDROID

Architecture, Diagnostic Réseau & Gestion de Scopesur Infrastructure Mobile Rootée

Document de portfolio techniqueSession de documentation — Août 2026Device : Xiaomi Redmi Note 10 5G (camellia)

TABLE DES MATIÈRES

1.  Résumé Exécutif  3

2.  Contexte & Architecture du Home-Lab  3

2.1  Device & Contraintes  3

2.2  Stack Opérationnelle  3

3.  Méthodologie de Diagnostic  4

3.1  Approche forensique  4

3.2  Chaîne de causalité  5

4.  Session de Résolution — Chronologie  5

4.1  Conflit VPN (InviZible vs Tailscale)  5

4.2  Clarification Tailscale / anonymisation  6

4.3  Tentative routage exit-node → Tor  6

4.4  Fixes secondaires  7

5.  Finding Technique : Exclusivité VPN Android  8

5.1  Le slot VpnService unique  8

5.2  Limitation userspace-networking  8

6.  Gestion Pragmatique du Scope  9

7.  État Actuel & Perspectives  10

8.  Glossaire  11

9.  Annexes  12

Annexe A : Commandes clés  12

Annexe B : Configuration torrc  12

Annexe C : Résolution AdGuard Home  13

10. Sources & Références  14

11. Annotations Personnelles  15


# 1. Résumé Exécutif


Ce document présente une session de diagnostic et d'architecture réseau menée sur un smartphone Android rooté (Xiaomi Redmi Note 10 5G) repurposé en serveur domestique multi-rôles. L'objectif était de résoudre une panne réseau affectant Tailscale, d'évaluer la faisabilité d'un routage anonymisé via Tor à travers un nœud de sortie Tailscale, et de stabiliser l'ensemble des services actifs (AdGuard Home, filebrowser, cloudflared, SSH).

Diagnostic root-cause réalisé via forensique processus/socket/iptables — aucune hypothèse non vérifiée n'a été retenue.

Conflit VPN résolu : InviZible Pro (mode VPN) et Tailscale se disputaient le slot VpnService unique d'Android ; élimination des règles iptables fantômes et du processus zombie pr0cks.

Finding technique documenté : l'implémentation userspace-networking de Tailscale sur Android rend l'interception iptables impossible, rendant le routage exit-node → Tor infaisable avec les outils actuels.

Gestion pragmatique du scope : abandon d'une voie technique (compilation Lokinet, module Magisk custom) jugée hors périmètre session, avec documentation complète des raisons.

Stabilisation des services secondaires : résurrection d'AdGuard Home bloqué par des processus zombies, correction de configuration SSH, création de scripts de persistance au démarrage.


# 2. Contexte & Architecture du Home-Lab



## 2.1  Device & Contraintes


Le device cible est un Xiaomi Redmi Note 5G (codename camellia) sous HyperOS/MIUI Android 13, rooté via Magisk. Avec environ 3,6 Go de RAM utilisable, il constitue une plateforme contrainte mais fonctionnelle pour un serveur domestique léger. Toutes les interactions sont réalisées à distance via SSH depuis un poste Windows 11 (Dell Latitude 5420) sous PowerShell, sans aucune interaction manuelle sur le téléphone — contrainte « zero-manual-interaction ».


## 2.2  Stack Opérationnelle


L'infrastructure repose sur Termux (avec Termux:Boot et tmux) comme couche d'exécution. Les services actifs et leur rôle sont résumés dans le tableau ci-dessous :

Service

Rôle

Statut Session

Termux + Termux:Boot + tmux

Couche d'exécution et persistance des sessions

Actif

Tailscale (Android app)

Mesh VPN privé, rôle exit-node sur tun0

Actif — 3 nœuds

AdGuard Home

Filtrage DNS local

Actif (résurrection post-reboot)

filebrowser

Partage de fichiers web

Actif

cloudflared

2× Quick Tunnels (accès externe)

Actif

Tor (Termux pkg)

Proxy SOCKS5 + TransPort (anonymisation)

Actif, standalone

llama.cpp

Inférence IA locale (GGUF)

Arrêté (contrainte RAM)

Kali NetHunter (chroot)

Outils de sécurité offensive

Présent, inactif

Magisk + AFWall+

Root + pare-feu iptables

Actif

InviZible Pro

VPN/Tor/DNSCrypt (précédemment)

Désinstallé de l'usage actif

Samba

Partage SMB

Désinstallé cette session


# 3. Méthodologie de Diagnostic


La philosophie adoptée tout au long de cette session repose sur un principe cardinal : « ne jamais supposer, toujours vérifier ». Chaque hypothèse a été soumise à une preuve empirique avant d'être retenue ou écartée.


## 3.1  Approche forensique : processus, sockets et iptables


Niveau 1 — Processus et sockets (userspace).

La première étape consiste à cartographier l'état du userspace : identification des processus actifs (ps, top), des sockets en écoute (ss -tlnp, netstat) et des fichiers ouverts (lsof, /proc/*/fd). Dans le cas présent, le processus pr0cks (composant proxy d'InviZible Pro, PID 32722) persistait malgré la fermeture forcée de l'application. Ce processus zombie, exécuté en root, maintenait un socket TCP bound sur 127.0.0.1:10080.

Niveau 2 — Règles de pare-feu (kernelspace).

L'inspection du kernelspace via iptables -t nat -L -n -v et iptables -t filter -L -n -v a révélé quatre règles fantômes : deux règles PREROUTING REDIRECT (tcp 80/443 → 127.0.0.1:10080) et deux règles FORWARD DROP sur tun0. Ces règles, bien que l'application source soit fermée, bloquaient tout trafic sortant de Tailscale.

Niveau 3 — Corrélation causale.

La corrélation entre le processus pr0cks (PID 32722), le socket 127.0.0.1:10080 et les règles iptables REDIRECT vers ce même port a établi sans ambiguïté la chaîne de causalité. Android n'autorisant qu'une seule interface VpnService (tun0) à la fois, InviZible Pro et Tailscale entraient en compétition pour ce slot exclusif.


## 3.2  Chaîne de causalité identifiée


Cause racine : InviZible Pro en mode VPN crée une interface tun0 et injecte des règles iptables REDIRECT/FORWARD.

Déclencheur : Tailscale tente de monter sa propre interface tun0 pour le rôle exit-node — échec silencieux ou partiel.

Effet persistant : Fermeture d'InviZible Pro ne supprime pas le processus pr0cks ni les règles iptables (pas de cleanup propre en root).

Symptôme : Tailscale apparemment « cassé » : pas de connectivité, nœuds inaccessibles.

Hypothèses écartées : Problème de clé WireGuard, conflit AFWall+, corruption de la base Tailscale — toutes vérifiées et infirmées.


# 4. Session de Résolution — Chronologie


Les événements de la session sont présentés dans l'ordre chronologique de leur occurrence. Chaque étape inclut l'action entreprise, la justification technique et le résultat observé.


## 4.1  Résolution du conflit VPN (InviZible Pro vs Tailscale)


Action 1 — Terminaison du processus zombie.

Le processus pr0cks (PID 32722) a été identifié via ps aux | grep pr0cks puis terminé par kill -9 32722. Un simple kill (SIGTERM) aurait pu suffire, mais l'état zombie du processus rendait un SIGKILL plus sûr.

Action 2 — Nettoyage des règles iptables fantômes.

Les quatre règles identifiées ont été supprimées via des commandes iptables -D ciblées : deux règles PREROUTING REDIRECT (tcp 80 et 443 vers 127.0.0.1:10080) et deux règles FORWARD DROP sur tun0. L'utilisation de -D avec une spécification exacte garantit la suppression précise sans risque de flush global affectant AFWall+.

Résultat.

Tailscale a immédiatement récupéré la connectivité. Le tailnet affiche désormais trois nœuds : redmi-note-10-5g (serveur, exit-node actif), redmi-note-14 (device secondaire) et tech/Windows laptop (poste de contrôle).


## 4.2  Clarification architecturale : Tailscale n'est pas un VPN anonymisant


Une misconception initiale a été corrigée : Tailscale n'anonymise pas l'adresse IP publique comme le ferait un VPN commercial. Tailscale est un réseau maillé privé (mesh VPN) basé sur WireGuard. Lorsqu'un nœud utilise un autre comme exit-node, le trafic sortant emprunte l'IP publique réelle du nœud de sortie — en l'occurrence, l'IP domestique. C'est un comportement par design, pas une limitation.

Implication.

Si l'objectif est l'anonymisation de l'IP publique, Tailscale seul est insuffisant. Une couche supplémentaire est nécessaire.


## 4.3  Tentative de routage exit-node → Tor


Face à la clarification ci-dessus, l'objectif a évolué : router le trafic sortant du nœud de sortie Tailscale à travers le réseau Tor pour masquer l'IP publique. Tor était déjà installé natif dans Termux (v0.4.9.11) et configuré avec SocksPort 9060 et TransPort 9061.

Vérification de Tor standalone.

Un test curl via le proxy SOCKS5 (127.0.0.1:9060) a confirmé le fonctionnement de Tor : la requête sortait bien par le réseau Tor avec une latence environ doublée par rapport au direct.

Tentative d'interception iptables.

Des règles iptables PREROUTING REDIRECT ont été créées pour rediriger le trafic tcp 80/443 entrant sur tun0 vers le TransPort Tor (127.0.0.1:9061). Malgré une navigation active à travers le nœud de sortie, les compteurs de paquets et d'octets de ces règles restaient à zéro. Le trafic ne traversait jamais la chaîne PREROUTING.

Analyse root-cause.

Recherche et confirmation : l'application Tailscale Android implémente le routage exit-node en userspace via gVisor netstack. Les connexions TCP/UDP sont terminées et ré-originaées en userspace, sans jamais passer par la pile netfilter du kernel. Par conséquent, iptables — qui opère au niveau du kernel — ne peut pas intercepter ce trafic. De plus, Tor est architecturalement incapable de transporter du trafic UDP (limitation protocolaire, pas une question de configuration).

Voies explorées et écartées.

• InviZible Pro en mode Root : déprécié depuis la bêta 1.6.7 ; seul le mode VPN reste disponible. Inapplicable.• Module Magisk magisk-tailscaled : exécute tailscaled avec -tun=userspace-networking, reproduisant le même problème. Non installé.• Lokinet : overlay packet-based supportant l'UDP, mais absence de package Termux/Android. Nécessite compilation from-source avec des capabilities setcap probablement bloquées par le sandbox Android. Hors scope session.

Décision.

Le routage Tailscale exit-node → Tor est abandonné comme infaisable avec l'outillage Android actuel. Les règles iptables REDIRECT mortes ont été supprimées. Tor reste installé et fonctionnel pour un usage manuel ou futur, mais n'est pas intégré au chemin de sortie.


## 4.4  Fixes Secondaires


Plusieurs problèmes secondaires ont été identifiés et résolus en parallèle :

Désinstallation de Samba. Le service Samba a été désinstallé à la demande de l'utilisateur. Le partage de fichiers est désormais entièrement délégué à filebrowser et aux tunnels cloudflared.

Résurrection d'AdGuard Home. Après un reboot, AdGuard Home refusait de démarrer avec l'erreur « opening db ... err=timeout ». L'inspection de /proc/*/cwd a révélé trois processus zombies AdGuardHome (invisibles à pgrep -x) détenant un verrou exclusif sur la base bbolt sessions.db. Un pkill -9 -f AdGuardHome a libéré le verrou, permettant un redémarrage propre dans une session tmux.

Vérification filebrowser & cloudflared. Les deux services ont été confirmés actifs post-reboot, bénéficiant de leur persistance via les sessions tmux existantes et Termux:Boot.

Persistance de Tor au démarrage. Un script de boot a été créé à ~/.termux/boot/start-tor (acquisition du wake-lock, délai de 5 secondes, lancement de tor), sur le modèle du script start-llama-server existant. Non testé sur un reboot réel à ce stade.

Correction SSH (Windows). Le fichier ~/.ssh/config du poste Windows référençait une IP locale obsolète (10.0.0.91) suite au changement d'IP du téléphone (10.0.0.72). Correction immédiate. Les trois alias SSH (redmi-local, redmi, redmi10) partagent une même paire de clés Ed25519 ; la distinction se fait par IP cible (locale vs Tailscale 100.75.47.3) et utilisateur (tech vs root). Pas de key-sprawl, juste une confusion d'alias.

llama.cpp — Dépriorisation. Le serveur d'inférence llama.cpp a été volontairement laissé arrêté cette session pour préserver la marge de RAM disponible. Ce n'est pas un échec mais une décision de priorisation des ressources.


# 5. Finding Technique : Contrainte d'Exclusivité VPN Android


Cette section formalise les deux découvertes techniques majeures de la session. Elles ne constituent pas des échecs mais des constats architecturaux documentés.


## 5.1  Le slot VpnService unique


Android, au niveau du framework, n'autorise qu'une seule interface VpnService active à la fois. Cette interface crée une interface tun0 dans le kernel, et toute application VPN doit acquérir ce slot. Lorsque deux applications tentent de l'occuper simultanément, la seconde échoue silencieusement ou provoque un comportement indéfini.

Conséquence pratique.

Il est impossible d'exécuter simultanément deux services VPN « classiques » sur Android sans une orchestration explicite. Cette contrainte n'est pas documentée de manière visible dans les interfaces utilisateur et constitue un piège fréquent pour les administrateurs système habitués aux environnements Linux où plusieurs interfaces tunnel coexistent naturellement.


## 5.2  Limitation userspace-networking de Tailscale


L'application Tailscale pour Android n'utilise pas le mode TUN kernel standard. Elle implémente son propre réseau virtuel en userspace via la bibliothèque gVisor netstack. Dans cette architecture, les paquets IP ne traversent pas la pile réseau du kernel Linux ; ils sont interceptés par le processus Tailscale, décapsulés, et les connexions TCP/UDP sont ré-originaées directement depuis l'espace utilisateur.

Impact sur iptables.

Puisque le trafic ne passe jamais par les hooks netfilter (PREROUTING, FORWARD, POSTROUTING), aucune règle iptables ne peut l'intercepter, le modifier ou le rediriger. Cela rend impossible : la redirection transparente vers un proxy local (Tor TransPort, Squid, etc.) ; le filtrage par iptables du trafic Tailscale ; toute forme de DPI ou de shaping au niveau kernel.

Impact sur Tor.

Même si l'interception iptables était possible, Tor présente une limitation architecturale fondamentale : il ne supporte que TCP. Tout trafic UDP (DNS, QUIC, STUN, etc.) serait silencieusement abandonné. Dans un contexte de nœud de sortie complet, cela briserait de nombreux services modernes.

Voie de contournement théorique.

La seule solution viable identifiée serait le développement d'une application Android VpnService personnalisée combinant un client Tailscale en mode kernel TUN avec un pont vers Tor. Cela représente un travail de plusieurs jours à plusieurs semaines (développement Kotlin/Java, gestion des permissions VpnService, compilation cross-platform de tailscaled, intégration Tor). Hors scope de la session présente, mais documenté comme piste future.


# 6. Gestion Pragmatique du Scope


Une compétence technique souvent sous-estimée est la capacité à reconnaître quand un « rabbit hole » dépasse la valeur apportée par sa résolution. Cette session en est un exemple concret.


## 6.1  Définition du rabbit hole


Le rabbit hole identifié est le routage anonymisé Tailscale → Tor sur Android. Les voies explorées ont toutes abouti à des impasses techniques ou à des charges de travail disproportionnées par rapport à l'objectif initial.


## 6.2  Critères d'abandon


Faisabilité technique : Aucune solution existante ne résout le problème avec l'outillage actuel.

Rapport effort/valeur : La compilation from-source de Lokinet ou le développement d'une app VpnService custom représentent des dizaines d'heures pour un bénéfice marginal.

Stabilité : Une solution hackée introduirait une dette technique et des risques de sécurité.

Objectif principal : Le cœur de la session était la résolution de la panne Tailscale et la stabilisation des services — objectifs atteints.


## 6.3  Leçon


Abandonner une voie technique n'est pas un échec lorsque l'abandon est documenté, justifié et que l'énergie est réaffectée vers des objectifs atteignables. La documentation du « pourquoi ça ne marche pas » est aussi précieuse que celle du « comment ça marche » — elle évite à l'équipe (ou au futur soi) de répéter les mêmes investigations infructueuses.


# 7. État Actuel & Perspectives



## 7.1  État opérationnel


Tailscale : mesh fonctionnel (3 nœuds), rôle exit-node actif, pas d'anonymisation IP.

Tor : installé et fonctionnel en mode standalone (SocksPort 9060, TransPort 9061), non intégré au chemin de sortie.

AdGuard Home : actif, filtrage DNS opérationnel.

filebrowser : actif, accès fichiers web fonctionnel.

cloudflared : 2× Quick Tunnels actifs.

llama.cpp : arrêté (décision de priorisation RAM).

Samba : désinstallé.

InviZible Pro : retiré de l'usage actif, aucun conflit résiduel.

Aucun conflit VPN/proxy résiduel détecté.


## 7.2  Items ouverts


Persistance Tor au boot : Le script ~/.termux/boot/start-tor est créé mais non testé sur un reboot réel. À valider.

Application VpnService custom : Piste identifiée pour résoudre le routage exit-node → Tor. Scope : plusieurs jours/semaines. Non engagé.

llama.cpp : Réactivation envisageable lorsque la charge RAM le permettra.


# 8. Glossaire


AFWall+ — Application Firewall pour Android. Pare-feu basé sur iptables avec interface graphique, nécessitant un accès root.

bbolt — Base de données clé-valeur embarquée en Go (BoltDB), utilisée par AdGuard Home pour stocker ses sessions et statistiques.

chroot — Change root. Mécanisme d'isolation de répertoire racine permettant d'exécuter un environnement Linux complet à l'intérieur d'Android.

cloudflared — Client officiel Cloudflare pour créer des tunnels sécurisés (Quick Tunnels) sans ouverture de ports sur le routeur.

exit-node — Nœud de sortie dans Tailscale. Tout le trafic Internet d'un device client peut être routé à travers ce nœud.

filebrowser — Application web de gestion de fichiers. Fournit une interface HTTP pour naviguer, télécharger et uploader des fichiers.

FORWARD DROP — Règle iptables bloquant le transit de paquets entre interfaces réseau.

gVisor netstack — Implémentation userspace de la pile réseau TCP/IP, développée par Google. Utilisée par Tailscale Android.

HyperOS/MIUI — Surcouche logicielle de Xiaomi basée sur Android. HyperOS est le successeur de MIUI.

iptables — Framework de filtrage de paquets du kernel Linux. Permet de définir des règles de pare-feu, de NAT et de redirection.

llama.cpp — Implémentation C/C++ optimisée pour l'inférence de modèles de langage (LLM) au format GGUF, conçue pour du matériel limité.

Lokinet — Réseau overlay décentralisé basé sur Oxen. Supporte TCP et UDP via un tunnel packet-based, contrairement à Tor.

Magisk — Solution de root pour Android moderne. Fonctionne par modification de la partition boot sans altération du système.

NAT REDIRECT — Règle iptables modifiant la destination d'un paquet à la volée, typiquement pour rediriger du trafic vers un proxy local.

PREROUTING — Chaîne iptables traitant les paquets entrants avant qu'ils ne soient routés.

pr0cks — Composant proxy d'InviZible Pro. Écoute sur un port local et redirige le trafic vers Tor/DNSCrypt.

Quick Tunnel — Tunnel temporaire créé par cloudflared sans nécessiter de compte Cloudflare ni de configuration DNS permanente.

SocksPort / TransPort — Ports d'écoute de Tor. SocksPort expose un proxy SOCKS5. TransPort permet la redirection transparente via iptables.

tailnet — Réseau privé créé par Tailscale. Ensemble des nœuds appartenant à un même utilisateur ou équipe.

Termux — Émulateur de terminal et environnement Linux pour Android. Permet d'installer et d'exécuter des packages natifs.

Termux:Boot — Extension Termux permettant l'exécution automatique de scripts au démarrage du device.

tun0 — Interface réseau virtuelle TUN (layer 3) créée par les applications VPN sous Android.

userspace-networking — Mode d'opération où la pile réseau est implémentée entièrement en espace utilisateur, sans passer par le kernel.

VpnService — API Android permettant à une application de créer une interface VPN tun0 et d'intercepter le trafic réseau du système.

WireGuard — Protocole VPN moderne, léger et performant, utilisé par Tailscale comme couche de transport chiffrée.


# 9. Annexes



## Annexe A : Commandes clés de la session


Diagnostic processus :ps aux | grep pr0cksss -tlnp | grep 10080iptables -t nat -L -n -viptables -t filter -L -n -v

Terminaison zombie :kill -9 32722

Nettoyage iptables :iptables -t nat -D PREROUTING ...  # spécification exacte de la règleiptables -t filter -D FORWARD ...  # spécification exacte de la règle

Inspection zombies AdGuard :for pid in /proc/[0-9]*; do readlink $pid/cwd; done | grep AdGuardHome

Libération verrou bbolt :pkill -9 -f AdGuardHome

Test Tor standalone :curl --socks5 127.0.0.1:9060 https://check.torproject.org


## Annexe B : Configuration torrc


SocksPort 127.0.0.1:9060TransPort 127.0.0.1:9061# DNSPort 127.0.0.1:9062  # envisagé, non activéRunAsDaemon 0DataDirectory /data/data/com.termux/files/home/.tor


## Annexe C : Résolution du blocage AdGuard Home


Symptôme : après reboot, AdGuard Home échoue au démarrage avec « opening db ... err=timeout ».Root cause : trois processus AdGuardHome zombies (non listés par pgrep -x) conservent un verrou exclusif flock sur la base bbolt sessions.db. La nouvelle instance ne peut pas acquérir le verrou et timeout.Résolution : inspection manuelle des répertoires de travail via /proc/*/cwd pour identifier les PIDs cachés, suivie d'un pkill -9 -f AdGuardHome. Redémarrage propre dans une session tmux dédiée.


# 10. Sources & Références


Tailscale Documentation — « How Tailscale works » (tailscale.com/kb/1151/what-is-tailscale)

Tailscale GitHub — « Android userspace-networking implementation » (github.com/tailscale/tailscale-android)

gVisor netstack — « Userspace TCP/IP stack » (github.com/google/gvisor/tree/master/pkg/tcpip)

Android Developers — « VpnService » (developer.android.com/reference/android/net/VpnService)

Tor Project — « TransPort and DNSPort limitations » (torproject.org/docs/tor-manual.html)

InviZible Pro GitHub — « Root mode deprecation notice » (github.com/Gedsh/InviZible)

Termux Wiki — « Running services in background » (wiki.termux.com)

Cloudflare Docs — « cloudflared Quick Tunnels » (developers.cloudflare.com/cloudflare-one/connections/connect-apps)

AdGuard Home — « Troubleshooting database locks » (github.com/AdguardTeam/AdGuardHome/wiki)


# 11. Annotations Personnelles


Cette session illustre pourquoi je privilégie systématiquement la méthodologie forensique aux hypothèses rapides. Le symptôme « Tailscale ne marche pas » aurait pu mener à une réinstallation complète, une régénération des clés WireGuard, ou pire, un flash système. En remontant la chaîne causale étape par étape — processus, sockets, iptables, framework Android — la vraie cause (un processus zombie et des règles fantômes) a été isolée en quelques minutes.Le finding sur userspace-networking est particulièrement important pour mon portfolio. Ce n'est pas une simple correction de configuration : c'est une compréhension profonde de l'architecture réseau d'Android et des compromis faits par Tailscale pour fonctionner dans un écosystème contraint. Savoir quand une technologie atteint ses limites fondamentales — et être capable de l'expliquer — distingue un technicien d'un ingénieur.La gestion du scope est tout aussi significative. Il est facile de se laisser entraîner dans une compilation from-source de Lokinet ou un développement d'application custom. Reconnaître que ces voies dépassent le ROI de la session, les documenter proprement, et réaffecter l'énergie vers des objectifs atteignables, est une discipline que je considère comme essentielle dans tout environnement de production.Enfin, la contrainte « zero-manual-interaction » a forcé une rigueur particulière : chaque action doit être scriptable, reproductible et idempotente. C'est exactement l'état d'esprit que j'applique à l'infrastructure cloud et aux pipelines CI/CD : l'automatisation n'est pas un luxe, c'est une exigence de fiabilité.
