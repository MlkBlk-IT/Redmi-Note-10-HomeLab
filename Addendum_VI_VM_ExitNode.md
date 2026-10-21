HOME-LAB ANDROID

ADDENDUM VI

VM ExitNode

Chaînage Tailscale → OpenVPN (Proton) sur serveur virtualisé Hyper-V

Document de portfolio technique · Addendum VI

Session — 15 août 2026

Hôte : Dell Latitude 5420 (Windows 11 Pro) · Hyperviseur : Hyper-V

VM cible : ExitNode — Windows Server 2025 (Desktop Experience)

Document confidentiel · Usage portfolio uniquement


# Préface


Ce document a été rédigé avec l'assistance d'une intelligence artificielle conversationnelle (Claude, Anthropic). L'IA a été utilisée pour deux fonctions distinctes durant la session technique et durant la production de ce livrable :

Recherche documentaire en temps réel : validation de comportements techniques (routage Windows, comportement du driver TAP-Windows6, limites du plan gratuit Proton VPN, bugs connus de Tailscale sur Windows) via recherche web, en date du 15 août 2026, afin d'éviter de s'appuyer sur des suppositions ou des connaissances potentiellement obsolètes.

Rédaction de la documentation : mise en forme, structuration et écriture de ce document de synthèse à partir des actions réellement exécutées durant la session, telles que dirigées et validées à chaque étape par l'auteur.

Toutes les décisions d'architecture, les choix de configuration, les validations de résultats et les commandes exécutées sur les systèmes ont été dirigées, revues et approuvées par l'auteur humain à chaque étape. L'IA n'a agi à aucun moment de façon autonome sur les systèmes ; chaque commande a été proposée puis exécutée manuellement, avec le résultat collé et vérifié avant de poursuivre — conformément à la méthodologie déjà établie dans les addendums précédents de ce projet.

L'usage de l'IA est documenté ici par souci de transparence et de traçabilité, dans le même esprit qui a guidé la documentation des échecs, des abandons et des décisions de scope tout au long du projet Home-Lab Android : ce qui a été fait, comment, et avec quels outils, doit pouvoir être retracé sans ambiguïté.


# Table des matières


Préface	2

1. Résumé de la session	4

2. Contexte et architecture cible	4

2.1  Chaîne visée	4

2.2  Infrastructure hôte	4

3. Travaux réalisés	5

3.1  Mise en réseau de la VM (ICS)	5

3.2  Adressage IP fixe de la VM	5

3.3  Mise à jour du système	6

3.4  Installation et connexion de Tailscale	6

3.5  Vérification du routage DNS	6

3.6  Installation d'OpenVPN et connexion à Proton VPN	7

3.7  Correction du routage — priorité d'interface	7

3.8  Validation end-to-end	8

4. Finding technique	8

5. État actuel et items ouverts	9

5.1  État opérationnel	9

5.2  Items ouverts pour une prochaine session	9

6. Annexes	10

7. Glossaire	11


# 1. Résumé de la session


Cette session documente la mise en place complète d'une chaîne de sortie Internet anonymisée sur une nouvelle brique d'infrastructure : une machine virtuelle Hyper-V dédiée (ExitNode, Windows Server 2025), hébergée sur le poste de contrôle habituel du projet (Dell Latitude 5420, Windows 11 Pro). L'objectif était de transférer le rôle d'exit-node Tailscale — jusqu'ici porté par le Redmi Note 10 5G — vers cette VM, et d'y chaîner un tunnel OpenVPN vers un serveur Proton VPN gratuit, de façon à ce que tout le trafic sortant de la VM (et donc de tout appareil du tailnet l'utilisant comme exit-node) sorte par une IP publique masquée.

Le résultat est un succès end-to-end vérifié depuis un appareil tiers du tailnet (Redmi Note 14) : la sortie Internet de ce device, en passant par l'exit-node ExitNode, est confirmée comme provenant du réseau Proton AG (New York), et non de l'adresse IP domestique.

📌 Contexte. Ce document est le sixième addendum du projet Home-Lab Android. Il s'appuie directement sur les diagnostics de routage établis dans l'Addendum II (conflit ip rule Tailscale/OpenVPN) et sur la découverte du userspace-networking de Tailscale documentée dans le Portfolio principal — deux limitations qui, sur Android, avaient empêché la même architecture d'aboutir sur ce système d'exploitation. Sur Windows Server, ces limitations ne s'appliquent pas de la même façon, ce qui a permis la réussite de cette session.


# 2. Contexte et architecture cible



## 2.1 Chaîne visée


La chaîne de bout en bout mise en place est la suivante :

Redmi Note 14 (device quotidien)   │  Tailscale (mesh WireGuard)   ▼ExitNode — Windows Server 2025 (VM Hyper-V, exit-node du tailnet)   │  OpenVPN (client) — TAP-Windows6   ▼Proton VPN — serveur US-FREE#120 (37.19.199.154)   │   ▼Internet — IP publique masquée (Proton AG, New York)

Le Redmi Note 10 5G, qui portait auparavant le rôle d'exit-node dans une architecture antérieure (voir Addendums II et III), conserve son rôle de serveur de résolution DNS (AdGuard Home) pour l'ensemble du tailnet. Il n'est plus dans le chemin de sortie du trafic Internet.


## 2.2 Infrastructure hôte


Élément

Détail

Hôte physique

Dell Latitude 5420, Windows 11 Pro, utilisateur tech

Hyperviseur

Hyper-V

VM cible

ExitNode (nom réel — distinct du nom d'affichage "WinServer-ExitNode" évoqué en phase de planification)

OS de la VM

Windows Server 2025, Desktop Experience

Mémoire

Dynamique — 2048 / 4096 / 8192 Mo (min / démarrage / max)

Disque

VHDX 40 Go

Réseau virtuel

vSwitch_WAN — commutateur interne avec partage de connexion Internet (ICS), sous-réseau 192.168.137.0/24

Routage/NAT natif

RRAS disponible nativement sur Windows Server 2025 mais non activé — ICS suffisant pour les besoins de sortie Internet de la VM. Capacité RRAS retenue pour une extension future du projet (plusieurs clients de sortie simultanés).

📌 Isolation confirmée vis-à-vis de l'autre lab. Le poste hôte héberge également un laboratoire de VM complet et indépendant, avec son propre serveur Windows Server 2025 faisant office de contrôleur DHCP pour ses machines. Ce lab utilise un commutateur virtuel Hyper-V distinct de vSwitch_WAN, sans aucune interface réseau partagée avec ExitNode. Aucun risque de collision d'adresse IP ou d'interférence de routage entre les deux environnements.


# 3. Travaux réalisés



## 3.1 Mise en réseau de la VM (ICS)


Deux causes distinctes ont été identifiées et corrigées avant que la VM obtienne un accès Internet fonctionnel :

Carte réseau non connectée à un commutateur virtuel : corrigée via Connect-VMNetworkAdapter, en associant l'adaptateur de ExitNode à vSwitch_WAN.

Partage de connexion Internet (ICS) cassé silencieusement par une adresse IP statique sur l'interface WiFi de l'hôte : corrigé en désactivant puis réactivant le partage de connexion via le panneau de propriétés réseau de Windows (ncpa.cpl), en confirmant explicitement la cible vSwitch_WAN.

Validation : la VM a reçu l'adresse 192.168.137.210/24 avec passerelle 192.168.137.1 via le DHCP intégré d'ICS. Les tests de connectivité (ping vers la passerelle et vers 8.8.8.8) ont montré 0 % de perte de paquets.

📌 Fausse piste écartée. Le blocage initial de trafic entrant par le pare-feu Windows de l'hôte (politique par défaut appliquée également sur une autre VM du parc, SRV1, sans effet observé sur son trafic ICS sortant) a été écarté comme cause possible après vérification croisée. Le trafic ICS sortant n'est pas affecté par les règles de blocage entrant.


## 3.2 Adressage IP fixe de la VM


Une adresse IP statique a été appliquée sur l'interface Ethernet de la VM (192.168.137.210/24, passerelle 192.168.137.1), reprenant l'adresse déjà attribuée dynamiquement par ICS, afin de garantir la stabilité de l'adressage pour les règles de pare-feu et la configuration réseau à venir. Aucune configuration DNS manuelle n'a été appliquée sur cette interface : la résolution DNS applicative de la VM est prise en charge séparément par Tailscale (voir section 3.5).


## 3.3 Mise à jour du système


Windows Update a été exécuté intégralement sur la VM avant toute installation logicielle, incluant des mises à jour de sécurité (.NET Framework, plateforme de sécurité Windows, définitions Microsoft Defender) ainsi qu'un redémarrage complet. La connectivité ICS a été revérifiée avec succès après redémarrage.


## 3.4 Installation et connexion de Tailscale


Le client Tailscale a été installé sur ExitNode et rattaché au même tailnet que les autres appareils du projet. La VM apparaît dans la console d'administration sous le nom exitnode, avec l'adresse tailnet 100.107.136.37.

Le tag Exit Node a été activé pour cette machine dans la console d'administration Tailscale, la rendant disponible comme point de sortie pour les autres appareils du réseau.

Machine

Adresse tailnet

Rôle

Statut

exitnode

100.107.136.37

Exit Node (nouveau)

Connecté

redmi-note-10-5g

100.75.47.3

Serveur DNS (AdGuard Home)

Connecté

redmi-note-14

100.122.0.101

Client (device quotidien)

Connecté

tech

100.116.22.124

Hors service — retiré du tailnet

Déconnecté (volontaire)

📌 Décision de scope — retrait de l'hôte. L'appareil tech (poste hôte physique) a été volontairement exclu du tailnet et de tout test d'exit-node. L'hôte héberge à la fois ExitNode (via ICS sur son interface WiFi) et l'autre laboratoire de VM du parc ; router son propre trafic à travers une VM qu'il héberge lui-même via ICS aurait introduit une dépendance circulaire de service, avec un risque de rupture de connectivité pour l'ensemble de l'infrastructure en cas de mauvaise interaction entre les tables de routage de Tailscale et le partage de connexion. Le rapport risque/valeur de ce test a été jugé défavorable et le test n'a pas été exécuté.


## 3.5 Vérification du routage DNS


Avant de poursuivre, il a été vérifié que la résolution DNS de la VM ExitNode continue de transiter par le serveur AdGuard Home hébergé sur le Redmi Note 10, et non par un résolveur générique. La commande suivante confirme que l'interface Tailscale de la VM pointe vers le résolveur interne de Tailscale (100.100.100.100), qui redirige lui-même vers le nameserver global configuré côté administration Tailscale :

Get-DnsClientServerAddress -InterfaceAlias "Tailscale*"# RésultatInterfaceAlias   ServerAddresses--------------   ---------------Tailscale        {100.100.100.100}

La console d'administration Tailscale (section Réseau → DNS) confirme la configuration du nameserver global : 100.75.47.3 (Redmi Note 10), avec l'option de recouvrement DNS (Override DNS servers) activée. Le schéma d'architecture DNS établi dans les addendums précédents reste donc intact malgré le changement de rôle d'exit-node.


## 3.6 Installation d'OpenVPN et connexion à Proton VPN


Le client OpenVPN GUI (édition communautaire open source, à ne pas confondre avec OpenVPN Connect, le client officiel d'OpenVPN Inc. qui ne supporte plus le pontage TAP) a été installé sur ExitNode. Le fichier de configuration .ovpn utilisé provient directement du compte Proton VPN existant du projet (compte gratuit, sans abonnement payant), téléchargé la veille de la session (garantissant sa compatibilité avec la politique de dépréciation des anciens fichiers de configuration manuelle appliquée par Proton VPN depuis le 28 février 2026).

Paramètre

Valeur

Fournisseur

Proton VPN — plan gratuit

Serveur

US-FREE#120 (37.19.199.154)

Protocole

OpenVPN sur UDP

Driver réseau

TAP-Windows6 (Adapter V9)

Adresse assignée par le tunnel

10.96.0.4

📌 Choix du driver TAP-Windows6. Le driver Wintun (partagé avec le mécanisme interne de Tailscale sous Windows) a été explicitement écarté au profit du driver historique TAP-Windows6, disponible en option lors de l'installation d'OpenVPN GUI. Ce choix évite un bug documenté de Tailscale sur Windows (issue GitHub tailscale/tailscale #7937) : lorsque le service Tailscale redémarre, il supprime tous les adaptateurs réseau de type Wintun présents sur la machine, y compris ceux utilisés par un autre logiciel — ce qui aurait cassé la connexion OpenVPN à chaque redémarrage du service Tailscale.


## 3.7 Correction du routage — priorité d'interface


Une fois le tunnel OpenVPN établi, la vérification de la table de routage a révélé que le trafic par défaut continuait de sortir par l'interface Ethernet (ICS), et non par le tunnel VPN, malgré la présence des routes de redirection globale (0.0.0.0/1 et 128.0.0.0/1) ajoutées par OpenVPN vers l'interface TAP.

# Avant correctionNetwork Destination   Netmask     Gateway           Interface          Metric0.0.0.0                0.0.0.0     192.168.137.1     192.168.137.210    150.0.0.0                128.0.0.0   10.96.0.1         10.96.0.4          281

La cause identifiée est un metric d'interface trop bas sur la carte Ethernet (15) par rapport au metric effectif de la route ajoutée par OpenVPN sur l'interface TAP (281) : Windows privilégie systématiquement la route au metric le plus faible, quelle que soit la spécificité de la route. Il s'agit d'un comportement documenté et récurrent d'OpenVPN sur Windows, indépendant de la présence ou non de la directive redirect-gateway def1 dans le fichier de configuration.

Correctif appliqué : augmentation du metric de l'interface Ethernet à 300, rendant la route par défaut via le tunnel TAP (metric 281) prioritaire.

Set-NetIPInterface -InterfaceAlias "Ethernet" -InterfaceMetric 300# Après correction0.0.0.0    0.0.0.0      192.168.137.1   192.168.137.210   3000.0.0.0    128.0.0.0    10.96.0.1       10.96.0.4         281   ← désormais prioritaire


## 3.8 Validation end-to-end


Le test final a été effectué en deux temps : d'abord localement sur ExitNode, puis depuis un appareil tiers du tailnet (Redmi Note 14) après sélection explicite de exitnode comme exit-node dans l'application Tailscale Android.

Point de test

IP publique observée

Organisation / lieu

Résultat

ExitNode (local, PowerShell)

159.26.98.241

AS208172 Proton AG, New York, US

Conforme

Redmi Note 14 (via tailnet)

159.26.98.241

AS208172 Proton AG, New York, US — détecté comme VPN par l'outil de test

Conforme, identique au test local

L'identité de l'adresse IP publique observée depuis les deux points de test confirme que l'intégralité du chemin réseau — Note14 → Tailscale → ExitNode → OpenVPN → Proton VPN → Internet — fonctionne comme prévu, sans fuite vers l'adresse IP domestique de l'utilisateur à aucune étape du parcours.


# 4. Finding technique : pourquoi cette architecture réussit sur Windows Server là où elle échouait sur Android


Les Addendums II et III de ce projet documentent en détail pourquoi la même intention architecturale — faire coexister Tailscale et un second tunnel VPN sur le même système, avec le second tunnel prioritaire pour la sortie Internet — avait échoué sur le Redmi Note 10 Android. Deux causes racines y avaient été identifiées : les règles de routage ip rule à haute priorité injectées par Tailscale (interceptant le trafic avant la table main où vivent les routes OpenVPN), et le fait que le client Tailscale Android route le trafic exit-node entièrement en espace utilisateur (gVisor netstack), sans jamais transiter par la pile réseau du noyau — rendant toute interception ou reroutage par les outils standard (iptables, ip rule) impossible.

Sur Windows Server, ces deux obstacles ne s'appliquent pas de la même manière :

Le client Tailscale pour Windows utilise le driver Wintun, un vrai adaptateur TUN au niveau noyau — pas une implémentation en espace utilisateur. Le trafic transite normalement par la pile réseau native de Windows, où le mécanisme standard de table de routage et de metric d'interface s'applique normalement.

Le mécanisme de conflit sur Windows n'est donc pas un verrou de priorité imposé par Tailscale (comme observé sur Android), mais un problème de priorité de route beaucoup plus classique et bien documenté : celui du metric d'interface entre deux adaptateurs réseau concurrents pour la route par défaut — un problème résolu en une seule commande une fois identifié.

📌 Portée de ce finding. Cette différence de comportement entre les deux plateformes n'était pas garantie a priori. Elle a été confirmée par recherche documentaire dédiée durant cette session (voir Annexe C), et non supposée par analogie avec l'architecture Android. Le mécanisme précis diffère selon le système d'exploitation : ce qui a échoué sur Android pour une raison architecturale profonde (userspace-networking) a réussi sur Windows Server pour une raison de configuration ordinaire (metric d'interface).


# 5. État actuel et items ouverts



## 5.1 État opérationnel


Composant

Détail

Statut

ExitNode — connectivité ICS

192.168.137.210/24, passerelle .137.1, IP fixée en statique

OK

ExitNode — Tailscale

100.107.136.37, tag Exit Node actif

OK

ExitNode — OpenVPN/Proton

Tunnel actif, driver TAP-Windows6, IP 10.96.0.4

OK

Routage prioritaire TAP

Metric Ethernet 300 > metric route TAP 281 — correction appliquée

OK

DNS du tailnet

Toujours routé vers Redmi Note 10 (AdGuard Home), non affecté par le changement d'exit-node

OK

Test end-to-end (Note14)

IP de sortie confirmée identique à ExitNode — Proton AG, New York

OK

Isolation pare-feu Windows sur ExitNode

Script prêt (blocage entrant par défaut, autorisation sortante, désactivation Bureau à distance) — non encore exécuté

TODO

Hôte (tech) dans le tailnet

Retiré volontairement, aucun test prévu

N/A

RRAS sur ExitNode

Non activé — capacité native disponible pour extension future (sortie multi-clients)

N/A


## 5.2 Items ouverts pour une prochaine session


Exécuter le script d'isolation pare-feu sur ExitNode (blocage entrant par défaut sur les trois profils, désactivation du groupe de règles Bureau à distance, sans modification des services TermService/fDenyTSConnections requis par le mode de session amélioré Hyper-V).

Évaluer la persistance du tunnel OpenVPN et de la priorité de metric après un redémarrage de la VM (le changement de metric appliqué via Set-NetIPInterface n'a pas été testé pour sa persistance au reboot).

Documenter et tester une procédure de reconnexion automatique du tunnel OpenVPN en cas de coupure (actuellement, la connexion a été établie manuellement via l'interface graphique OpenVPN GUI).

Envisager, en session dédiée future, l'activation du rôle RRAS sur ExitNode si le projet évolue vers un scénario de sortie Internet pour plusieurs clients simultanés au-delà du tailnet actuel.


# 6. Annexes



## Annexe A — Commandes clés de la session


# Rattacher la carte reseau de la VM au commutateur virtuelConnect-VMNetworkAdapter -VMName ExitNode -SwitchName vSwitch_WAN# Verifier l'etat de l'interface DHCP recue via ICSipconfig /all# Fixer une adresse IP statiqueNew-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.137.210 `  -PrefixLength 24 -DefaultGateway 192.168.137.1# Verifier le DNS pousse par TailscaleGet-DnsClientServerAddress -InterfaceAlias "Tailscale*"# Identifier le driver de l'adaptateur OpenVPNGet-NetAdapter | Where-Object {$_.InterfaceDescription -match "TAP|Wintun"}# Table de routage IPv4 completeroute print -4# Corriger la priorite de route (metric d'interface)Set-NetIPInterface -InterfaceAlias "Ethernet" -InterfaceMetric 300# Verifier l'IP de sortie publique et son organisationcurl.exe ifconfig.mecurl.exe ipinfo.io/<ip>


## Annexe B — Script d'isolation pare-feu (préparé, non exécuté)


Set-NetFirewallProfile -Profile Domain,Public,Private `  -DefaultInboundAction Block -DefaultOutboundAction AllowDisable-NetFirewallRule -DisplayGroup "Remote Desktop"# Ne pas modifier TermService ni fDenyTSConnections :# requis pour le mode de session ameliore Hyper-V (VMBus),# non lie au reseau, compatible avec l'isolation ci-dessus.


## Annexe C — Sources consultées durant la session


Tailscale Docs — « Exit nodes (route all traffic) » — tailscale.com/docs/features/exit-nodes

Tailscale Docs — « Can I use Tailscale alongside other VPNs? » — tailscale.com/docs/reference/faq/other-vpns

GitHub tailscale/tailscale, issue #7937 — « Tailscale removes all wintun adapters on service restart, not just its own »

Proton VPN — « How to download Proton VPN OpenVPN configuration files » — protonvpn.com/support/vpn-config-download

Proton VPN — « You may need to update your old OpenVPN configs » — protonvpn.com/blog/old-ovpn-configs

OpenVPN Community — téléchargements officiels et documentation du client OpenVPN GUI — openvpn.net/community-downloads/

OpenVPN Support Forum — plusieurs fils de discussion documentant le problème de metric d'interface entre TAP et carte réseau physique sous Windows

ipinfo.io — service de résolution d'organisation/ASN utilisé pour la vérification finale de l'IP de sortie


## Annexe D — Historique des versions du document


Version

Date

Modifications

1.0

15 août 2026

Création initiale — clôture de la session VM ExitNode


# 7. Glossaire


Exit node — Dans Tailscale, appareil du réseau maillé (tailnet) désigné pour faire transiter tout le trafic Internet des autres appareils qui le sélectionnent. Le trafic sortant emprunte alors l'adresse IP publique de cet appareil.

ICS (Internet Connection Sharing) — Fonctionnalité native de Windows permettant de partager la connexion Internet d'une interface réseau vers une autre, incluant un serveur DHCP minimal. Utilisée ici pour donner l'accès Internet à la VM via l'interface WiFi de l'hôte.

Metric (d'interface / de route) — Valeur numérique utilisée par Windows pour départager plusieurs routes possibles vers une même destination ; la route au metric le plus bas est préférée. Le metric effectif d'une route combine le metric de l'interface et le metric propre à la route.

OpenVPN GUI — Client OpenVPN open source pour Windows, maintenu par la communauté du projet OpenVPN. Distinct d'OpenVPN Connect, le client officiel commercial d'OpenVPN Inc., qui ne supporte plus le pontage TAP.

RRAS (Routing and Remote Access Service) — Rôle serveur natif de Windows Server permettant le routage IP complet, le NAT et l'hébergement de serveurs VPN entrants pour plusieurs clients. Disponible mais non activé dans cette session.

Tailnet — Réseau privé virtuel créé par Tailscale, regroupant l'ensemble des appareils d'un même compte utilisateur, chacun identifié par une adresse IP fixe dans la plage 100.64.0.0/10.

TAP-Windows6 — Driver d'adaptateur réseau virtuel de niveau 2/3 historiquement utilisé par OpenVPN sous Windows, distinct du driver Wintun plus récent partagé par Tailscale et les versions modernes d'OpenVPN.

vSwitch (commutateur virtuel Hyper-V) — Composant logiciel Hyper-V simulant un commutateur réseau physique, permettant à une ou plusieurs VM de communiquer entre elles et/ou avec le réseau physique de l'hôte, selon son type (Externe, Interne, Privé).

Wintun — Driver d'adaptateur réseau virtuel de niveau 3 développé par les créateurs de WireGuard, utilisé nativement par Tailscale sous Windows et disponible en option pour OpenVPN 2.5 et versions ultérieures.
