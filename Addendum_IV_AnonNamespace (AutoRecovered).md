HOME-LAB ANDROID

Addendum IV

Création et validation d'un Network Namespace isolé (anon_ns)

Document de portfolio technique · Addendum III

Session — 14 août 2026simbadug@

Device : Xiaomi Redmi Note 10 5G (camellia)


# Table des matières


Table des matières1

1. Résumé de la session1

2. Contexte1

3. Travaux réalisés1

3.1 Persistance de la mémoire virtuelle (zram + swapfile)1

3.2 Gestion de la charge batterie1

3.3 Correction du script Create-AnonNamespace.ps11

3.4 Exécution et résultats1

4. Vérifications post-implémentation1

5. État du système en fin de session1

6. Risques et limites identifiés1

7. Prochaines étapes1

Annexe A — Commandes clés de vérification1

Annexe B — Extrait de configuration réseau confirmée1

Sources1


# 1. Résumé de la session


Cette session avait pour objectif de tester la création d'un network namespace Linux isolé (anon_ns) sur le Redmi Note 10 5G, en mode test contrôlé : sans activation du VPN ni du firewall associé, dans le seul but de valider que la primitive d'isolation réseau fonctionne sur ce device et n'interfère pas avec le tunnel Tailscale déjà actif.

Un correctif préalable a également été appliqué à la gestion de la mémoire virtuelle (zram et swapfile), avec persistance au démarrage, ainsi que l'installation d'une application de gestion de charge de la batterie (BCL) suite à un blocage rencontré avec une première application (AccA) causé par une régression connue de Magisk 30.x.

Résultat global : toutes les vérifications post-implémentation sont positives. Le namespace a été créé, isolé, puis démonté sans effet résiduel. Tailscale est resté fonctionnel tout au long du test, confirmé par trois indicateurs indépendants (tables de routage, règles ip rule, process actif). Aucune modification n'a été persistée au démarrage — un redémarrage du téléphone effacerait entièrement le namespace, la configuration kernel n'ayant subi aucune modification permanente.


# 2. Contexte


Le projet Home-Lab Android vise depuis plusieurs sessions à isoler un tunnel VPN (OpenVPN, serveurs VPNGate) du trafic Tailscale existant. L'Addendum II avait établi le diagnostic root-cause : Tailscale injecte des règles de routage policy (ip rule) à haute priorité (13000-17000) qui interceptent le trafic avant que les routes OpenVPN de la table main ne soient consultées. La solution architecturale retenue est l'utilisation de network namespaces Linux pour donner au VPN une stack réseau totalement indépendante de celle utilisée par Tailscale.

Un script PowerShell (Create-AnonNamespace.ps1) avait été préparé lors d'une session antérieure mais jamais exécuté. Cette session avait pour objectif de le corriger, le valider, puis l'exécuter en mode test restreint (sans VPN ni firewall) afin de confirmer la viabilité de l'approche avant d'investir du temps dans la configuration complète.


# 3. Travaux réalisés



## 3.1 Persistance de la mémoire virtuelle (zram + swapfile)


Suite à une réinstallation complète du téléphone, la configuration de mémoire virtuelle personnalisée avait été perdue. L'état retrouvé montrait zram à sa priorité par défaut (-2) et un nouveau swapfile de 6 Go créé en priorité 10 — un ordre inversé par rapport à la configuration voulue.

Correction appliquée : zram reconfiguré en priorité 100 (prioritaire sur le swapfile), confirmé via /proc/swaps. Un script de démarrage a ensuite été créé et déployé (50-swap-persist.sh) pour garantir que cette configuration survive à un redémarrage : le script réapplique la priorité zram, puis recrée le swapfile de 6 Go uniquement s'il est absent — sans jamais écraser un fichier existant — avant de l'activer en priorité 10.

Validation : le script a été exécuté manuellement (hors reboot) et a confirmé son idempotence — sur un swapfile déjà présent, la création est correctement sautée.


## 3.2 Gestion de la charge batterie


Une première application de limitation de charge (AccA, via F-Droid) a été installée mais n'a jamais pu obtenir les droits root : aucune fenêtre de confirmation Magisk n'apparaissait à l'ouverture de l'application. Le diagnostic par analyse des logs système (logcat) a révélé une exception interne au processus Magisk (android.system.ErrnoException: stat failed: ENOENT), survenant avant même que l'application ne tente de contacter le démon root — confirmant une régression connue de Magisk 30.1 et versions ultérieures (documentée publiquement, issues GitHub #9206 et #9570), indépendante de toute erreur de configuration locale.

Une application alternative (BCL — Battery Charge Limiter, fork maintenu par MuntashirAkon, package io.github.muntashirakon.bcl) a été installée via transfert direct de l'APK et pm install, contournant l'interface F-Droid. Contrairement à AccA, BCL a correctement obtenu les droits root. Un script de démarrage (51-bcl-boot.sh) a été déployé pour lancer l'application automatiquement au boot, celle-ci ne proposant pas de mécanisme de démarrage automatique natif.


## 3.3 Correction du script Create-AnonNamespace.ps1


Le script préparé lors d'une session antérieure contenait plusieurs valeurs de configuration non finalisées (adresse IP hôte, serveur VPN cible) ainsi qu'un défaut structurel : les commandes réseau nécessitant les privilèges root étaient envoyées directement sans passer par su -c, alors que l'utilisateur SSH configuré (tech) ne dispose pas de ces privilèges par défaut.

Corrections apportées :

Adresse IP de connexion fixée à 10.0.0.72 (LAN), après avoir confirmé par un test dédié que l'accès root direct via SSH (root@10.0.0.72) ne fournissait en réalité qu'un contexte utilisateur sandboxé (uid=10260, untrusted_app), non un accès root réel

Élévation de privilèges centralisée : chaque commande passe désormais par su -c via la fonction Invoke-AndroidCommand, avec échappement des guillemets pour éviter les conflits de quoting déjà documentés dans les addendums précédents

VPN et firewall (étapes 8 et 9 du script original) rendus conditionnels et désactivés pour ce test, afin d'isoler strictement la validation de la création du namespace

Ajout de deux vérifications automatiques de l'état de Tailscale — une avant toute modification, une après — pour détecter immédiatement toute régression


## 3.4 Exécution et résultats


Le script a été exécuté par copier-coller dans une session PowerShell interactive (contrainte locale : la politique d'exécution PowerShell bloquait le lancement direct de fichiers .ps1 ; corrigée en cours de session via Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser). Les packages requis (openvpn, root-repo, dnsmasq) ont été installés avec succès, suivis de la création du namespace anon_ns et de sa paire d'interfaces veth.


# 4. Vérifications post-implémentation


Après exécution du script, six vérifications indépendantes ont été effectuées manuellement pour confirmer l'absence d'impact sur l'infrastructure existante, en particulier sur le tunnel Tailscale actif.

Élément vérifié

Constat

État

Namespace anon_ns

Créé, veth0/veth1 up, IPs 10.200.1.1 / 10.200.1.2 assignées

OK

Tables de routage hôte (ip route show table all)

Table tun1 (Tailscale) intacte avec ses 4 IPs. Seule addition : 10.200.1.0/24 dev veth0 dans la table main, isolée

OK

Règles ip rule (priorités 13000-17000)

Identiques à l'état pré-namespace. Aucune règle liée à anon_ns détectée

OK

Process Tailscale (com.tailscale.ipn)

Actif, état sleeping normal (S), PID stable, aucun crash

OK

iptables nat / FORWARD sur l'hôte

Aucune règle ajoutée (étapes 8-9 du script intentionnellement désactivées)

OK

Persistance au boot (~/.termux/boot/*.sh)

Aucune trace de anon_ns dans les scripts de démarrage — namespace non persisté

OK

Point méthodologique important : l'interface tailscale0, recherchée dans une première vérification automatisée, n'existe pas sur l'implémentation Android de Tailscale. Le client Android utilise un réseau virtuel géré entièrement en espace utilisateur (gVisor netstack, tel que documenté dans le Portfolio Home-Lab Android section 5.2) et expose ses routes sous le nom d'interface tun1, non tailscale0. La vérification correcte — confirmée dans cette session — repose sur trois signaux concordants : la présence de la table de routage tun1 avec ses IPs, l'absence de modification des règles ip rule 13000-17000, et l'état actif du process com.tailscale.ipn.


# 5. État du système en fin de session


Composant

Détail

Statut

Namespace anon_ns

Créé, actif en mémoire, non persisté au boot

TEMP

Tailscale (mesh + exit-node)

Actif, aucune interférence détectée

OK

zram

Priorité 100, persistance au boot via 50-swap-persist.sh

OK

Swapfile

6 Go, priorité 10, persistance au boot

OK

BCL (gestion batterie)

Installée, root fonctionnel, lancement au boot configuré

OK

AccA (abandonnée)

Root non fonctionnel — régression Magisk 30.x confirmée

N/A

VPN / Firewall namespace

Non configurés — étape volontairement exclue de ce test

TODO


# 6. Risques et limites identifiés


Le namespace anon_ns et sa paire veth sont des structures kernel volatiles : elles ne survivent pas à un redémarrage du téléphone. Aucun flash, aucune modification de kernel ou de partition système n'a été effectué — un reboot ramènerait le device à son état réseau par défaut sans aucune action corrective requise.

Le namespace est actuellement vide : aucune route de sortie utile (pas de VPN, pas de règles firewall). Il ne remplit pas encore sa fonction cible et nécessite une session dédiée pour l'étape suivante.

Le script exécute une connexion SSH séparée par commande (une quinzaine d'allers-retours pour ce test), ce qui a nécessité la saisie répétée du mot de passe SSH en l'absence d'authentification par clé configurée pour l'utilisateur tech.


# 7. Prochaines étapes


Configurer une authentification par clé SSH pour l'utilisateur tech afin d'éliminer la saisie répétée du mot de passe lors de l'exécution de scripts multi-commandes

Choisir un serveur VPNGate cible et renseigner son IP dans la configuration du script (variable VpnServerIP)

Activer le mode VPN/Firewall du script (EnableVpnFirewall = $true) et relancer la création du namespace avec le tunnel OpenVPN et les règles de firewall default-deny

Valider le changement d'IP publique depuis l'intérieur du namespace (curl ifconfig.me via ip netns exec)

Envisager la persistance du namespace au boot via un script Termux:Boot dédié, une fois la configuration VPN validée et jugée stable


# Annexe A — Commandes clés de vérification


Vérifier l'état du namespace et ses interfaces :

ssh redmi10 "su -c 'ip netns list; ip netns exec anon_ns ip addr show'"

Vérifier les tables de routage complètes de l'hôte :

ssh redmi10 "su -c 'ip route show table all'"

Vérifier les règles de routage policy (RPDB) :

ssh redmi10 "su -c 'ip rule show'"

Vérifier l'activité du process Tailscale :

ssh redmi10 "su -c 'ps -A'" | Select-String -Pattern "tailscale"

Rollback complet du namespace (suppression) :

ssh redmi10 "su -c 'ip netns del anon_ns; ip link del veth0 2>/dev/null; true'"


# Annexe B — Extrait de configuration réseau confirmée


Table de routage tun1 (Tailscale), inchangée avant/après création du namespace :

100.75.47.3      dev tun1  table tun1  proto static  scope link100.100.100.100  dev tun1  table tun1  proto static  scope link100.116.22.124   dev tun1  table tun1  proto static  scope link100.122.0.101    dev tun1  table tun1  proto static  scope link

Seule addition observée dans la table main :

10.200.1.0/24 dev veth0 proto kernel scope link src 10.200.1.1


# Sources


Addendum II — Home-Lab Android : Routage Kernel, Conflit ip rule & Network Namespaces (session Août 2026)

Portfolio Home-Lab Android : Architecture, Diagnostic Réseau & Gestion de Scope (section 5.2 — Limitation userspace-networking de Tailscale)

GitHub Magisk — Issues #9206 et #9570 (régression de la fenêtre de confirmation root, versions 30.1+)

Documentation Tailscale — absence de support CLI natif sur Android (tailscale.com/docs/reference/tailscale-cli)
