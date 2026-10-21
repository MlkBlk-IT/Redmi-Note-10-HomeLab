Redmi Note 10 — Diagnostic et remise en état réseau

Panne hotspot, nettoyage Termux/Tor, configuration AdGuardHome + Tailscale

14 août 2026


# 1. Le problème de départ


Le hotspot WiFi du Redmi Note 10 ne fonctionnait plus. Le téléphone tournait aussi plusieurs services en arrière-plan via Termux (SSH, un serveur IA local llama-server, un tunnel Cloudflare, un serveur de fichiers, Tor, et AdGuardHome comme bloqueur de pub/DNS), démarrés automatiquement via Termux:Boot.


# 2. Diagnostic — pourquoi le hotspot ne démarrait pas


Premier symptôme : le toggle du hotspot Android se remettait à OFF tout seul en une seconde.

En inspectant les journaux système Android (logcat) au moment précis de l'activation du hotspot, l'erreur suivante est apparue :

E Tethering: Error in setDnsForwarders

Cette erreur signifie qu'Android n'arrive pas à configurer le serveur DNS interne (dnsmasq) nécessaire pour distribuer Internet aux appareils connectés au hotspot. Sans DNS fonctionnel, Android considère que tout le tethering a échoué et coupe l'interface immédiatement.

Cause n°1 identifiée : AdGuardHome écoutait sur le port 53 (le port DNS standard) sur toutes les interfaces réseau du téléphone (0.0.0.0:53), y compris celle utilisée par le hotspot. Le service système dnsmasq ne pouvait donc pas prendre ce port, d'où le message d'erreur :

E dnsmasq : failed to create listening socket: Address already in use

Après avoir libéré le port 53, le hotspot a fonctionné une première fois — confirmant le diagnostic.

Cause n°2 identifiée (après reconfiguration d'AdGuardHome) : le hotspot recommençait à échouer dès que Tailscale (VPN) était actif. Un VPN Android capture tout le trafic réseau du système, y compris le DNS, ce qui empêche le service de tethering natif de fonctionner correctement en parallèle. C'est un conflit connu et documenté entre les VPN Android et le tethering natif.


# 3. Nettoyage effectué



## 3.1 Tor — supprimé complètement


Processus Tor actif tué en mémoire

Paquet Tor désinstallé de Termux (pkg uninstall)

Fichiers de configuration résiduels supprimés (torrc, tor.log)

Script de démarrage automatique (start-tor) supprimé du dossier .termux/boot/

L'application InviZible Pro (pan.alexander.tordnscrypt.stable), un second composant lié à Tor/DNSCrypt installé séparément, désinstallation en cours


## 3.2 llama-server (serveur IA local) — désactivé au démarrage


Script de démarrage automatique renommé en 20-llama-server.disabled (réversible, rien n'est supprimé)

Processus actif tué en mémoire (via la session tmux redmi-llama)


## 3.3 AdGuardHome — reconfiguré, pas désactivé


Objectif : garder AdGuardHome comme résolveur DNS pour les appareils connectés via Tailscale, sans qu'il bloque le port 53 pour le hotspot.

Le fichier de configuration (AdGuardHome.yaml) a été modifié : le DNS (port 53) écoute désormais uniquement sur l'adresse Tailscale du téléphone (100.75.47.3), au lieu de toutes les interfaces (0.0.0.0)

Le tableau de bord web (port 3000) reste accessible normalement sur le réseau local, aucun changement

Le service a été relancé pour appliquer la config, puis vérifié : DNS actif sur 100.75.47.3:53 uniquement

Le nameserver 100.75.47.3 a été ajouté dans la console d'administration Tailscale (DNS → Nameservers), avec l'option "Override DNS servers" activée, pour que tous les appareils du réseau Tailscale utilisent AdGuardHome automatiquement

Vérification finale : tableau de bord accessible, 22 019 requêtes DNS traitées, 98,37 % du trafic provenant bien du sous-réseau Tailscale (100.75.x.x)


## 3.4 Watchdog AdGuardHome — remis en place


Le script 40-adguard.sh (qui relance AdGuardHome automatiquement s'il s'arrête) avait été désactivé temporairement pendant le diagnostic, puis réactivé à la demande une fois le conflit de port résolu.


# 4. Hotspot + Tailscale en simultané — la solution retenue


Le hotspot natif Android ne peut pas fonctionner en même temps qu'un VPN actif (Tailscale) à cause du conflit DNS décrit plus haut, même après la reconfiguration d'AdGuardHome. Deux solutions ont été envisagées :

Fermer Tailscale le temps d'utiliser le hotspot, puis le rallumer — solution simple et fiable, mais peu pratique

Utiliser l'application VPNHotspot pour partager la connexion VPN via le hotspot — solution technique retenue

Le premier essai avec le hotspot Android natif + VPNHotspot en support a échoué ("Tether internal failure") car le hotspot système plante avant même que l'application puisse intervenir.

Solution qui a fonctionné : le mode Repeater de VPNHotspot, qui crée son propre point d'accès WiFi via Wi-Fi Direct, indépendamment du service de tethering Android cassé. Ce mode contourne complètement le bug setDnsForwarders.


## 4.1 Incident — WiFi complet bloqué


Lors d'un test ultérieur, l'utilisation du mode Repeater a fait planter complètement l'interface WiFi du téléphone (wlan0), pas seulement le hotspot : le WiFi normal est resté grisé et impossible à réactiver dans les paramètres.

Cause probable : le mode Repeater bascule le chipset WiFi en mode point d'accès (Wi-Fi Direct), ce qui peut désactiver le WiFi client normal — et il n'est pas toujours revenu correctement à la normale après désactivation.

Résolution : un redémarrage complet du téléphone a rétabli le WiFi normalement. Point à retenir : le mode Repeater de VPNHotspot doit être utilisé avec prudence — vérifier ses réglages (mode "safe" vs "unsafe") avant la prochaine utilisation prolongée.


# 5. Nettoyage des applications (debloat)


Une liste des applications tierces installées a été passée en revue. Décision prise :

InviZible Pro (pan.alexander.tordnscrypt.stable) — à supprimer (résidu Tor)

AdAway — conservé (bloqueur de pub redondant avec AdGuardHome, mais gardé à la demande)

AFWall+ (pare-feu tiers) — conservé (utile pour surveiller le réseau)

La suppression d'InviZible Pro a rencontré une erreur système (bug du shell pm sur cette version MIUI) et reste à finaliser.


# 6. État final du système


Service

État final

Démarre au boot

Tor (Termux + app InviZible Pro)

Supprimé

Non

llama-server

Désactivé

Non

AdGuardHome (DNS)

Actif — Tailscale uniquement

Oui

AdGuardHome (dashboard web)

Actif — accessible réseau local

Oui

cloudflared

Actif

Oui

filebrowser

Actif

Oui

sshd

Actif

Oui

Hotspot WiFi

Fonctionnel via VPNHotspot (mode Repeater)

Manuel


# 7. Points de vigilance pour la suite


Le mode Repeater de VPNHotspot peut bloquer le WiFi normal — à utiliser avec prudence, vérifier les réglages avant réutilisation prolongée

Si l'IP Tailscale du téléphone change un jour (100.75.47.3), la configuration d'AdGuardHome (AdGuardHome.yaml) et le nameserver dans la console Tailscale devront être mis à jour en conséquence, sinon AdGuardHome cessera de démarrer correctement

Finaliser la désinstallation d'InviZible Pro (pan.alexander.tordnscrypt.stable)

Le script 40-adguard.sh contient toujours une ligne qui tue le service dnsmasq système (pkill dnsmasq) à chaque relance — sans risque tant qu'AdGuardHome reste limité à l'interface Tailscale, mais à garder en tête
