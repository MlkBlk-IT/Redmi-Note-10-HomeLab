Home-Lab Android · Addendum VI

ADDENDUM VI

Bascule du Tunnel OpenVPN vers anon_ns : Isolation Réussie vs. Portée Réelle du Test

Suite aux Addendums I–V, Home-Lab Android

Document de portfolio technique · Addendum VI

Session — 14–15 août 2026

Device : Xiaomi Redmi Note 10 5G (camellia)


# 1. Résumé de la session


Cette session fait suite directe à l'Addendum V, qui avait laissé ouverte la question du mécanisme exact bloquant le trafic retour dans le network namespace anon_ns (ICMP et négociation TLS/UDP d'OpenVPN échouant systématiquement à l'intérieur du namespace, alors que fonctionnels hors namespace). L'objectif de cette session était de contourner ce blocage par une approche différente : plutôt que de faire négocier OpenVPN à l'intérieur d'anon_ns, établir le tunnel OpenVPN dans le host (où il fonctionne sans obstacle), puis déplacer l'interface tun0 déjà active vers anon_ns une fois la négociation TLS complétée.

Cette approche s'est révélée concluante. Le tunnel a été établi en foreground une première fois pour diagnostic (confirmant Initialization Sequence Completed), puis relancé en mode daemon, puis son interface tun0 déplacée avec succès dans anon_ns via ip link set tun0 netns anon_ns. Après correction de deux effets de bord du déplacement (perte de l'adresse IPv4 sur l'interface, route par défaut résiduelle pointant vers l'ancien veth), le trafic sortant d'anon_ns a été confirmé traverser le tunnel OpenVPN : curl -4 ifconfig.me depuis le namespace retourne 159.26.98.239 (IP du serveur Proton VPN ciblé), et le ping vers 9.9.9.9 réussit à 0% de perte.

Un second temps de la session a porté sur la clarification de la portée exacte de ce résultat. Une confusion initiale a émergé sur ce que le test avait réellement validé : la vérification portait sur des commandes exécutées via ip netns exec anon_ns depuis une session SSH root, pas sur le trafic des applications Android normales (navigateur, etc.) tournant sur le device. Cette distinction a nécessité plusieurs échanges pour être établie clairement, aboutissant à deux constats structurants pour la suite du projet : (1) un network namespace kernel est invisible pour les applications Android standard, qui ne peuvent pas y être placées sans mécanisme dédié ; (2) faire sortir le trafic Tailscale lui-même (et donc les appareils distants du tailnet utilisant le Redmi Note 10 comme exit-node) à travers ce tunnel n'est structurellement pas possible avec l'outillage actuel, pour la même raison userspace-networking déjà documentée dans le Portfolio principal — confirmé de nouveau cette session par une recherche ciblée sur la documentation Tailscale à jour.


# 2. Contexte et objectif de la session


L'Addendum V avait laissé le projet dans un état de diagnostic avancé mais non résolu : bpftool avait permis de cartographier les programmes eBPF actifs et d'éliminer plusieurs hypothèses (netd, xt_bpf, AdGuard Home/AdAway), sans identifier le mécanisme exact bloquant le trafic retour dans anon_ns. Le blocage touchait spécifiquement ICMP (raw socket) et la négociation UDP bidirectionnelle d'OpenVPN — confirmé par le test décisif où le même fichier .ovpn négociait en moins d'une seconde hors namespace, contre un échec systématique par timeout à l'intérieur.

Décision de contournement. Plutôt que de poursuivre l'investigation eBPF (bloquée par l'absence de CONFIG_DEBUG_INFO_BTF sur ce kernel, comme documenté en Addendum V section 6.2), la session a adopté la voie pragmatique déjà identifiée en fin d'Addendum V : établir OpenVPN hors du namespace où il fonctionne sans obstacle, puis migrer uniquement l'interface tun0 déjà négociée vers anon_ns. Cette approche évite complètement la question du blocage de négociation, puisque la négociation TLS/UDP n'a jamais lieu à l'intérieur du namespace — seul le trafic de données post-négociation y transite.


# 3. Logique technique de l'approche retenue



## 3.1 Pourquoi déplacer une interface plutôt que négocier dans le namespace


Un network namespace Linux isole une stack réseau complète, mais les interfaces réseau elles-mêmes (tun0, veth, etc.) peuvent être créées dans un namespace puis déplacées vers un autre via ip link set <iface> netns <namespace>. Ce mécanisme est indépendant de l'état de la connexion portée par l'interface : une fois qu'OpenVPN a complété sa négociation TLS et établi le tunnel de données chiffré sur tun0, l'interface elle-même devient un objet kernel simple (une paire de tampons de paquets liés au tunnel chiffré), déplaçable comme n'importe quelle interface réseau.

Le processus OpenVPN lui-même reste rattaché au host — son socket UDP vers le serveur VPN (37.19.199.154) continue d'exister dans la stack réseau du host, hors d'anon_ns. Ce qui change de contexte réseau, c'est uniquement l'interface tun0 : les paquets IP applicatifs qui y entrent (depuis anon_ns) sont chiffrés par le processus OpenVPN côté host et envoyés au serveur VPN via le socket UDP du host — un socket qui, lui, n'a jamais eu à négocier depuis l'intérieur du namespace bloqué.

Distinction clé. Le blocage caractérisé en Addendum V portait sur la capacité d'anon_ns à mener une négociation réseau bidirectionnelle complète (handshake TLS, échange ICMP). En déplaçant une interface déjà négociée, la session contourne cette limitation sans la résoudre : le tunnel n'a jamais eu à négocier depuis l'intérieur du namespace. La cause exacte du blocage originel reste donc non identifiée — mais elle devient sans conséquence pour l'objectif opérationnel (faire sortir le trafic d'anon_ns par un VPN isolé de Tailscale).


## 3.2 Effets de bord attendus d'un déplacement d'interface


Le déplacement d'une interface vers un autre namespace ne préserve pas automatiquement toute sa configuration. Deux effets ont été anticipés et confirmés lors de l'exécution :

Adresses IP non préservées : les adresses IPv4/IPv6 assignées à l'interface dans le namespace d'origine ne sont pas automatiquement recréées dans le namespace de destination. Seule une adresse IPv6 link-local est régénérée automatiquement par le kernel (spécifique à chaque namespace).

Routes non préservées : les routes associées à l'interface (route par défaut via tun0, route vers le sous-réseau assigné) n'existent que dans la table de routage du namespace d'origine ; le namespace de destination possède sa propre table de routage, déjà peuplée de ses propres routes par défaut (ici, la route héritée de la configuration veth d'anon_ns).

Ces deux points impliquaient qu'une reconfiguration manuelle serait nécessaire immédiatement après le déplacement — anticipation confirmée par les résultats du test (section 4.3).


# 4. Séquence de tests et résultats



## 4.1 Diagnostic préalable : environnement host


Avant de procéder au déplacement, un état des lieux du firewall et des permissions du device a été effectué pour écarter toute cause de blocage supplémentaire au niveau host.

Firewall host (iptables filter INPUT/FORWARD/OUTPUT) : policy ACCEPT sur les trois chaînes, aucune règle de blocage active rencontrée sur le chemin testé.

Règles FORWARD spécifiques à veth0 (interface d'anon_ns) déjà en place : ACCEPT bidirectionnel entre veth0 et wlan0, avec suivi d'état RELATED,ESTABLISHED — cohérent avec la configuration Addendum III.

NAT POSTROUTING : règle MASQUERADE active pour 10.200.1.0/24 (sous-réseau anon_ns) sortant vers wlan0 — confirmant que la sortie NAT standard d'anon_ns vers Internet direct (hors VPN) est fonctionnelle et non bloquée.

Permissions /dev/net/tun : accessible en root (uid=0), contexte SELinux u:r:magisk:s0, mode Enforcing actif — sans effet bloquant démontré, la création manuelle d'une interface tun de test (testtun) ayant réussi sans erreur.

Firewall interne d'anon_ns (namespace du test) : policy DROP par défaut sur INPUT/FORWARD/OUTPUT, avec une liste explicite de règles ACCEPT pour lo, tun0, l'état RELATED,ESTABLISHED, le port DNS 53 vers 9.9.9.9, et les cinq ports de fallback du serveur VPN (80, 4569, 1194, 51820, 5060 vers 37.19.199.154 en UDP) — confirmant le comportement default-deny documenté en Addendum III comme volontaire.

Ce diagnostic a permis d'établir que le firewall host n'était pas en cause dans le blocage caractérisé en Addendum V, et que l'environnement était prêt pour un test de déplacement d'interface sans obstacle identifié au niveau iptables/SELinux.


## 4.2 Établissement du tunnel OpenVPN hors namespace


Un premier test en foreground (mode --verb 4, non-daemon, avec limite de 25 secondes) a confirmé la négociation complète :

Initialization Sequence Completed

Data Channel: cipher 'AES-256-GCM', peer-id: 65548

Timers: ping 10, ping-restart 60

Le processus a été interrompu par le timeout de 25 secondes (code de sortie 124), comportement attendu et non révélateur d'un problème — le tunnel avait déjà terminé sa négociation en moins d'une seconde après le lancement, confirmé par l'horodatage des logs.

Le tunnel a ensuite été relancé en mode --daemon avec fichier de log dédié. Après un délai de stabilisation de 6 secondes, l'inspection a confirmé : processus openvpn actif (PID stable), interface tun0 présente avec le flag UP,LOWER_UP, et log confirmant de nouveau Initialization Sequence Completed.


## 4.3 Déplacement de tun0 vers anon_ns et corrections associées


Le déplacement initial (ip link set tun0 netns anon_ns) a réussi immédiatement — confirmé par la disparition de l'interface du host (Device "tun0" does not exist lors d'une nouvelle interrogation côté host) et son apparition dans anon_ns, mais à l'état DOWN. Un premier test de connectivité a échoué (ping: sendmsg: Operation not permitted, 100% de perte), cohérent avec les effets de bord anticipés en section 3.2 : la route par défaut d'anon_ns pointait toujours vers l'ancien veth1, et l'interface elle-même n'était pas encore activée dans son nouveau contexte.

Trois corrections séquentielles ont été appliquées :

Activation de l'interface : ip netns exec anon_ns ip link set tun0 up. Résultat : l'interface passe à l'état UP,LOWER_UP, mais seule une adresse IPv6 link-local (fe80::...) est présente — aucune adresse IPv4, confirmant la perte d'adressage anticipée.

Correction de la route par défaut : suppression de la route par défaut résiduelle (via veth1) et ajout d'une nouvelle route par défaut directement via tun0. Résultat : la table de routage d'anon_ns pointe désormais correctement vers tun0, mais le test de connectivité reste négatif — absence d'adresse IPv4 sur l'interface empêchant tout trafic sortant utile.

Réassignation de l'adresse IPv4 : ip netns exec anon_ns ip addr add 10.96.0.17/16 dev tun0 — reprise de l'adresse observée dans les logs OpenVPN originaux (assignée par le serveur Proton lors de la négociation). Cette étape a débloqué la connectivité.


## 4.4 Confirmation finale


Après application des trois corrections, les tests de validation ont été répétés avec succès :

curl -4 ifconfig.me  →  159.26.98.239

ping -c 2 9.9.9.9    →  2 paquets transmis, 2 reçus, 0% perte

rtt min/avg/max/mdev = 20.530/20.793/21.057/0.300 ms

L'adresse 159.26.98.239 correspond à l'IP publique du serveur Proton VPN ciblé (us120), distincte de l'IP domestique du Redmi Note 10 et de celle de l'exit-node Tailscale. Le test confirme que le trafic émis depuis l'intérieur d'anon_ns (via ip netns exec) traverse effectivement le tunnel OpenVPN plutôt que de sortir directement par le WiFi ou par Tailscale.


# 5. Clarification de la portée du résultat


Un second temps de la session, distinct du travail technique de déplacement d'interface, a porté sur la clarification de ce que ce résultat couvre réellement — clarification rendue nécessaire par une confusion initiale sur la portée du test.


## 5.1 Ce qui a été validé : isolation réseau au niveau process


Le test confirme qu'un process explicitement lancé avec le préfixe ip netns exec anon_ns hérite de la stack réseau isolée et sort par le tunnel OpenVPN. Cela couvre tout binaire exécutable depuis un shell root sur le device : outils Termux (curl, wget, nmap, ssh), le chroot Kali NetHunter dans son ensemble s'il est lancé avec ce préfixe, ou tout process daemon démarré de la même façon.


## 5.2 Ce qui n'a pas été validé : trafic applicatif Android natif


Une application Android standard (navigateur Chrome, ou toute autre app) ne s'exécute jamais avec ip netns exec — ce mécanisme n'existe pas au niveau du framework applicatif Android. Une app tourne dans le namespace réseau par défaut du système, complètement étanche d'anon_ns. Par conséquent, tester la connectivité via ip netns exec depuis une session SSH root ne dit rien sur le comportement du navigateur ou de toute autre app installée sur le device — ces derniers continuent, sans changement, de sortir par la route réseau par défaut (Tailscale exit-node ou WiFi direct selon la configuration active).

Implication pour l'objectif « faire sortir Chrome par le VPN ». Ce résultat ne le permet pas tel quel. Le mécanisme standard Android pour capturer le trafic de toutes les apps du device (y compris le navigateur) est l'API VpnService, utilisée par les apps VPN classiques (OpenVPN Connect, OpenVPN for Android/ics-openvpn) et par Tailscale lui-même. Un network namespace kernel, bien qu'efficace pour isoler des process shell, n'est pas ce mécanisme et ne s'y substitue pas.


## 5.3 Piste alternative identifiée : proxy local exposé depuis anon_ns


Une voie intermédiaire a été identifiée en fin de session, non testée : faire tourner un serveur proxy (SOCKS5 ou HTTP) directement à l'intérieur d'anon_ns, donc sortant par tun0/OpenVPN, puis exposer ce proxy vers le host via une redirection de port. Une application comme Chrome pourrait alors être configurée manuellement (paramètres de proxy) pour utiliser ce point d'entrée — sans que Chrome lui-même tourne dans le namespace. Cette piste reste à documenter et tester dans une session dédiée ; elle n'a fait l'objet d'aucune implémentation cette session.


## 5.4 Vérification documentaire : redirection du trafic Tailscale lui-même


Une question distincte a émergé en fin de session : la documentation Tailscale à jour permettrait-elle de rediriger le trafic de Tailscale lui-même (et donc, par extension, celui d'appareils tiers du tailnet comme un second device utilisant le Redmi Note 10 comme exit-node) vers un proxy ou tunnel externe ? Une vérification ciblée a été effectuée sur la documentation Tailscale et plusieurs sources tierces.

Constat. La fonctionnalité --socks5-server documentée par Tailscale (mode userspace networking) expose un proxy SOCKS5/HTTP local permettant à d'autres applications d'entrer dans le tailnet — c'est un point d'entrée vers le réseau Tailscale, pas un point de sortie vers un tunnel externe. Elle ne redirige pas le trafic sortant normal des apps à travers un exit-node puis vers un VPN tiers. Des projets tiers non officiels (par exemple TailSocks) permettent d'exposer un proxy SOCKS5 local routant vers un exit-node Tailscale choisi, mais aucun mécanisme, officiel ou tiers, ne permet l'opération inverse recherchée : faire sortir le trafic Tailscale lui-même vers un tunnel externe.

Ce constat confirme et renforce le finding déjà documenté dans le Portfolio principal (section 5.2) : l'implémentation Android de Tailscale opère en userspace-networking via gVisor netstack, sans jamais transiter par la pile netfilter du kernel — la rendant non interceptable par les mécanismes kernel-level (iptables, network namespaces) utilisés dans ce projet. La vérification documentaire de cette session n'a trouvé aucun développement récent contredisant ce finding.


# 6. Synthèse



## 6.1 Résultat technique


La bascule d'une interface OpenVPN déjà négociée vers anon_ns constitue une solution opérationnelle validée au problème de routage isolé identifié depuis l'Addendum II : un tunnel VPN dédié, dont le trafic ne transite jamais par les règles ip rule à haute priorité de Tailscale (Addendum II), fonctionne pour tout process explicitement exécuté dans le namespace. Le contournement évite la question non résolue du blocage de négociation caractérisé en Addendum V, sans la trancher — la cause exacte de ce blocage demeure non identifiée.


## 6.2 Portée réelle vs. portée visée


Objectif

Statut

Détail

Process shell isolé sortant par VPN dédié

VALIDÉ

Confirmé par ip netns exec, curl et ping

Chroot Kali isolé sortant par VPN dédié

VALIDÉ (par extension)

Même mécanisme que tout process anon_ns, non re-testé explicitement

Navigateur Android (Chrome) sortant par VPN

NON VALIDÉ

Nécessite VpnService, hors mécanisme netns

Device entier (toutes apps) sortant par VPN

NON VALIDÉ

Nécessite app VPN classique, incompatible avec Tailscale actif simultanément

Appareil tiers du tailnet (ex. Redmi Note 14) sortant via ce VPN

NON TESTÉ — piste théorique

Nécessiterait rediriger le trafic entrant par l'exit-node Tailscale vers anon_ns — pas la même contrainte que rediriger Tailscale lui-même

Trafic Tailscale lui-même redirigé vers un tunnel externe

CONFIRMÉ INFAISABLE

userspace-networking / gVisor, vérifié de nouveau sur documentation à jour


## 6.3 Ce qui reste ouvert


Proxy SOCKS5/HTTP exposé depuis anon_ns vers le host, pour permettre à des apps configurées manuellement (Chrome inclus) d'utiliser le tunnel sans en faire partie au niveau namespace — piste identifiée, non testée.

Redirection du trafic entrant via l'exit-node Tailscale (provenant d'appareils tiers du tailnet) vers anon_ns au niveau du routage/NAT host — mécanisme distinct de la redirection de Tailscale lui-même, potentiellement faisable car ce trafic transite déjà par le kernel une fois reçu via l'exit-node, mais non scopé ni testé cette session.

Persistance au boot du tunnel OpenVPN + déplacement netns : la séquence actuelle est entièrement manuelle et volatile (non testée après reboot, cohérent avec l'état TEMP déjà noté en Addendum III pour le namespace lui-même).

Script de rollback/cleanup pour remettre le device dans un état propre entre les sessions de test — identifié comme prochaine tâche opérationnelle, non encore réalisé.


# 7. Annexes



## Annexe A — Séquence de commandes clés (ordre d'exécution)


Lancement du tunnel OpenVPN en mode daemon (host) :

openvpn --config us120.ovpn --auth-user-pass us120.auth --daemon --writepid ovpn.pid --log ovpn.log --tmp-dir <tmp> --dev tun0 --verb 3

Déplacement de l'interface vers anon_ns :

ip link set tun0 netns anon_ns

Activation et correction du routage dans le namespace :

ip netns exec anon_ns ip link set tun0 up

ip netns exec anon_ns ip route del default

ip netns exec anon_ns ip route add default dev tun0

Réassignation de l'adresse IPv4 (reprise depuis les logs OpenVPN) :

ip netns exec anon_ns ip addr add 10.96.0.17/16 dev tun0

Validation finale :

ip netns exec anon_ns curl -4 -s --max-time 10 ifconfig.me

ip netns exec anon_ns ping -c 2 9.9.9.9


## Annexe B — Méthode de transfert des commandes (rappel)


Conformément à la méthode zero-nesting établie dans l'Addendum, chaque séquence de commandes a été encodée en base64 côté PowerShell local avant transmission SSH, évitant tout conflit de quoting imbriqué à travers les couches PowerShell → SSH → su :

$b64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($cmd)); ssh -p 8022 tech@10.0.0.72 "su -c 'echo $b64 | base64 -d | sh'"


## Annexe C — État du firewall anon_ns observé (extrait)


Chain OUTPUT (policy DROP)

ACCEPT udp -- 0.0.0.0/0 -> 37.19.199.154 udp dpt:80

ACCEPT udp -- 0.0.0.0/0 -> 37.19.199.154 udp dpt:4569

ACCEPT udp -- 0.0.0.0/0 -> 37.19.199.154 udp dpt:1194

ACCEPT udp -- 0.0.0.0/0 -> 37.19.199.154 udp dpt:51820

ACCEPT udp -- 0.0.0.0/0 -> 37.19.199.154 udp dpt:5060

ACCEPT all  -- * tun0


# 8. Sources


Tailscale Docs — « Userspace networking mode (for containers) » (tailscale.com/docs/concepts/userspace-networking)

GitHub — tailscale/tailscale, Issue #12933, « Add HTTP/SOCKS proxy support for running Tailscale »

GitHub — ItalyPaleAle/tailsocks, « Route traffic through Tailscale exit nodes using a local SOCKS5 proxy »

Linux man pages — ip-link(8), ip-netns(8), ip-route(8)

OpenVPN Reference Manual — openvpn.net/community-resources/reference-manual-for-openvpn-2-6/

Addendum II — Routage Kernel, Conflit ip rule & Network Namespaces

Addendum III — Création et validation d'un Network Namespace isolé (anon_ns)

Addendum V — Diagnostic Network Namespace : eBPF, cgroup-BPF & Isolation ICMP

Portfolio Home-Lab Android — section 5.2, Limitation userspace-networking de Tailscale


# 9. Annotations personnelles


Cette session illustre une distinction méthodologique utile : contourner un problème n'est pas la même chose que le résoudre, et les deux ont leur place selon l'objectif. L'Addendum V cherchait la cause racine du blocage de négociation dans anon_ns et s'est arrêté faute d'outillage (BTF absent). Cette session a atteint l'objectif opérationnel — un tunnel isolé fonctionnel — sans jamais trancher cette cause racine. C'est un choix assumé : la valeur recherchée était le résultat (isolation VPN fonctionnelle), pas la compréhension exhaustive du mécanisme de blocage. Les deux objectifs sont légitimes, mais les confondre aurait pu faire perdre une session entière à poursuivre un diagnostic devenu secondaire.

Le moment le plus important de la session n'est pas technique mais méthodologique : la clarification de portée en section 5. Un test réussi (curl retourne bien l'IP du VPN) peut être décrit de façon exacte tout en étant compris de façon inexacte si le contexte d'exécution n'est pas explicite. « Le trafic sort par le VPN » est vrai pour un process anon_ns et faux pour Chrome — la phrase seule ne le précise pas. Documenter cette distinction ici évite qu'un futur soi (ou un lecteur du portfolio) ne suppose, à tort, que la bascule de tun0 a résolu l'objectif plus large de changer l'IP publique du device en usage quotidien.

La vérification documentaire sur Tailscale et les proxys SOCKS5 mérite d'être notée comme bon réflexe plutôt que comme détour : une question légitime (« la doc a-t-elle changé depuis le dernier finding ? ») a été vérifiée par la preuve — recherche ciblée — plutôt que laissée sur la base d'un souvenir du Portfolio principal, qui datait de plusieurs sessions. Le résultat confirme le finding existant, mais la vérification elle-même était la bonne démarche, indépendamment de son issue.

Enfin, cette session referme un chapitre technique ouvert depuis l'Addendum II (routage isolé pour VPN dédié) tout en ouvrant clairement le suivant : la question de la portée device-entier ou tailnet-entier n'est plus une confusion mais une piste scopée (proxy exposé, redirection du trafic exit-node entrant). C'est la progression attendue d'un projet de home-lab documenté correctement — chaque session referme une question précise et documente la suivante sans la mélanger avec la précédente.

Document confidentiel · Usage portfolio uniquement
