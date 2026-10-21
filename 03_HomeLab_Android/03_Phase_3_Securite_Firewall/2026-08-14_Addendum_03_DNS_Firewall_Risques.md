Home-Lab Android · Addendum III

ADDENDUM III

DNS Filtrant, Mini-Firewall Kernel & Gestion de Risque de Démonstration

Suite aux documents Home-Lab Android (Portfolio, Addendum I, Addendum II)

Document de portfolio technique · Addendum III

Session — Août 2026

Device : Xiaomi Redmi Note 10 5G (camellian, variant Global/EEA)


# TABLE DES MATIÈRES



# 1. Résumé de la session


Cet addendum III documente une session de conception architecturale (sans exécution sur le device) portant sur trois axes : le durcissement du network namespace anon_ns introduit dans l'Addendum II, la gestion du risque opérationnel lié à la dépendance à un fournisseur VPN gratuit (VPNGate) dans un contexte de démonstration académique, et la validation de la disponibilité des paquets Termux requis.

Contrairement aux sessions précédentes, cette session s'est déroulée entièrement en phase de discussion et de conception — aucune commande n'a été exécutée sur le Redmi Note 10 ce soir. L'objectif était de préparer une architecture robuste avant une démonstration académique prévue dans un délai de deux semaines, en anticipant les points de rupture plutôt qu'en les découvrant en conditions réelles.

Un changement de direction majeur a également été acté au début de la session : l'abandon complet du chroot Kali NetHunter (désinstallé) au profit d'une architecture reposant uniquement sur le kernel Android natif. Cette décision élimine une couche d'abstraction entière et simplifie la chaîne de dépendances documentée dans l'Addendum I.


# 2. Changement d'architecture : abandon du chroot Kali


Le chroot Kali NetHunter (/data/local/nhsystem/kali-arm64), central dans les Addendums I et II, a été supprimé. La justification est double : la complexité de maintenance qu'il ajoutait (inventaire de paquets, PATH fix systématique, couche de transfert zero-nesting dédiée) et le constat que l'objectif réel — isoler un tunnel VPN du reste du trafic — ne nécessite aucun outil offensif ni environnement Debian complet.

Conséquence directe. Le network namespace anon_ns, dont le principe a été validé dans l'Addendum II comme piste théorique, devient l'unique mécanisme d'isolation. Tous les processus (OpenVPN, resolver DNS) y sont lancés directement comme binaires Termux natifs, sans chroot intermédiaire.


# 3. Vérification de disponibilité des paquets Termux


Avant toute implémentation, la disponibilité des deux binaires requis a été vérifiée par recherche plutôt que supposée — cohérent avec la règle de vérification systématique déjà appliquée dans ce projet.

Paquet

Dépôt

Commande d'installation

Statut

openvpn

Dépôt principal Termux

pkg update && pkg install openvpn

Confirmé disponible

dnsmasq

Dépôt root-repo (séparé)

pkg install root-repo && pkg install dnsmasq

Confirmé disponible — root requis

Note. dnsmasq est catégorisé comme paquet root-only côté Termux (repo termux-root-packages), ce qui est cohérent avec le fait que le device est déjà rooté et administré via su. Aucune surprise ni blocage identifié à ce niveau.


# 4. Architecture réseau cible : anon_ns durci



## 4.1  Principe général


L'architecture retient le network namespace anon_ns (créé via ip netns add, relié à l'hôte par une paire veth, avec NAT MASQUERADE sortant) comme fondation, telle que scriptée dans Create-AnonNamespace.ps1. Deux ajouts majeurs ont été apportés ce soir : un resolver DNS dédié au namespace et un mini-firewall interne.


## 4.2  DNS dédié — pas de double filtrage


Un point technique a été clarifié : faire tourner une seconde instance AdGuard Home à l'intérieur d'anon_ns a été jugé inutile, voire contre-productif. AdGuard Home continue de servir la résolution filtrée normale du lab (port 5353, routé depuis Tailscale/tun1), un rôle qui reste inchangé.

Dans anon_ns, un simple forwarder DNS (dnsmasq, sans filtrage applicatif) relaie les requêtes vers un résolveur externe fixe et connu pour son filtrage de sécurité — Quad9 (9.9.9.9) a été retenu comme upstream par défaut, en dur dans la configuration plutôt que dépendant du push DHCP-option du serveur OpenVPN, dont l'application n'est pas garantie hors client Windows/Android natif.

Justification du choix. Le trafic destiné à anon_ns n'a pas vocation à être filtré par les blocklists personnalisées d'AdGuard — c'est un chemin de test isolé. Superposer un second AdGuard ajouterait de la latence et un point de panne supplémentaire pour un bénéfice nul dans ce contexte.


## 4.3  Mini-firewall default-deny propre au namespace


Un constat de sécurité a motivé cet ajout : AFWall+ (l'application firewall du device, documentée dans le Portfolio principal) pilote iptables uniquement dans le namespace réseau par défaut d'Android. Un nouveau network namespace démarre avec une table iptables vide et totalement indépendante — AFWall+ n'a donc aucune visibilité ni contrôle sur anon_ns.

Risque identifié. Sans règles explicites, anon_ns constitue une sortie réseau non filtrée, non auditée par la couche de sécurité applicative existante — un chemin qu'un processus compromis pourrait exploiter pour exfiltrer du trafic sans passer par les protections habituelles du device.

Mitigation retenue. Un jeu de règles iptables minimal est appliqué directement dans anon_ns via ip netns exec, en politique default-deny : tout trafic sortant est bloqué par défaut, à l'exception de trois exceptions explicites — loopback, le resolver DNS filtrant (port 53 vers l'IP Quad9 uniquement), et le serveur VPN cible (IP et port précis). Une fois le tunnel OpenVPN monté, le trafic via l'interface tun0 est également autorisé.

ip netns exec anon_ns iptables -P OUTPUT DROP

ip netns exec anon_ns iptables -P INPUT DROP

ip netns exec anon_ns iptables -A OUTPUT -o lo -j ACCEPT

ip netns exec anon_ns iptables -A OUTPUT -p udp -d 9.9.9.9 --dport 53 -j ACCEPT

ip netns exec anon_ns iptables -A OUTPUT -p <proto> -d <IP_VPN> --dport <port> -j ACCEPT

ip netns exec anon_ns iptables -A OUTPUT -o tun0 -j ACCEPT

ip netns exec anon_ns iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

Ce mini-firewall doit être appliqué immédiatement après la création du namespace et avant le lancement de tout processus applicatif (dnsmasq, openvpn), afin d'éviter une fenêtre de non-filtrage.


# 5. Script mis à jour : Create-AnonNamespace.ps1


Le script PowerShell de création du namespace a été étendu à 10 étapes pour intégrer l'installation des paquets, le DNS filtrant et le mini-firewall. Résumé des ajouts par rapport à la version initiale :

Étape 0 — installation idempotente de openvpn et dnsmasq (root-repo)

Étapes 1 à 8 — inchangées : vérifications kernel, création namespace, paire veth, configuration IP, forwarding, NAT

Étape 9 — application du mini-firewall default-deny (flush préalable, politique DROP, whitelist DNS + serveur VPN + tun0)

Vérifications finales — test positif (ping vers le DNS filtré, doit réussir) et test négatif (ping vers une IP non whitelistée, doit échouer) — preuve empirique du bon fonctionnement du firewall plutôt qu'une simple supposition

Le script reste paramétré par variables en tête de fichier (IP du serveur VPN, port, protocole, DNS filtrant), permettant un changement de fournisseur VPN en modifiant deux ou trois valeurs plutôt qu'en réécrivant la logique.


# 6. Gestion de risque — Démonstration académique



## 6.1  Le problème identifié


La session a soulevé un risque opérationnel concret : une démonstration académique prévue dans environ deux semaines dépend, dans l'architecture actuelle, d'un serveur VPNGate précis (IP figée dans la configuration ET dans la règle de whitelist du firewall). VPNGate étant un service bénévole et communautaire, la disponibilité d'un serveur donné n'est pas garantie dans le temps — un serveur fonctionnel aujourd'hui peut disparaître avant la date de présentation.

Aggravant identifié. Le mini-firewall default-deny introduit en §4.3, bien que bénéfique pour la sécurité, ajoute un second point de fragilité : même en changeant manuellement de serveur VPNGate, l'ancienne IP resterait seule autorisée tant que le firewall n'est pas régénéré avec la nouvelle cible.


## 6.2  Stratégie retenue


Plutôt que de développer un mécanisme de fallback automatique (bascule scriptée entre plusieurs serveurs VPNGate), une approche de gestion de risque pragmatique a été retenue, cohérente avec la méthodologie de gestion de scope déjà appliquée dans les documents précédents :

Démonstration principale : un fournisseur VPN commercial stable (Proton VPN, offre gratuite) sera utilisé pour la présentation devant le professeur — élimine le risque de serveur mort en cours de démonstration.

Démonstration bonus : si le temps le permet, une bascule vers VPNGate sera démontrée en direct, illustrant que l'architecture est agnostique au fournisseur VPN — seules les variables IP/port/protocole du script changent, pas la logique.

Filet de sécurité pratique : en cas d'usage de VPNGate, retélécharger le CSV et sélectionner un serveur vivant le jour même de la démonstration plutôt que de réutiliser une IP validée des semaines à l'avance.

Ce choix illustre un principe déjà présent dans les annotations personnelles des addendums précédents : documenter une dépendance fragile et la contourner par une décision de scope est aussi valable, pour une évaluation académique, que de sur-ingénierer un mécanisme de fallback automatique dont la complexité dépasserait le bénéfice réel pour ce contexte.


## 6.3  Estimation de performance (contexte : test de traçage de paquets, non temps réel)


Une estimation théorique de la perte de débit cumulée sur la chaîne complète (Note 14 → Tailscale → Note 10 → anon_ns → OpenVPN → Internet) a été établie à titre indicatif :

Couche

Perte de débit estimée

Impact latence

Tailscale (WireGuard)

~5-10 %

Faible (quelques ms, réseau local)

veth / namespace

~1-3 %

Négligeable (kernel-to-kernel)

OpenVPN (chiffrement)

~15-25 %

Variable selon distance serveur

iptables (double couche)

Négligeable

Négligeable

Conclusion retenue. La perte de débit cumulée (~25-35 %) est jugée non pertinente pour l'usage visé — un test de traçage de paquets (vérification de sortie via ifconfig.me / whatsmyip) ne dépend ni du débit ni de la latence. La distance géographique du serveur VPN choisi n'a donc aucune incidence sur la validité du test, seulement sur le confort d'usage, qui n'est pas un critère pour cette démonstration.


# 7. Risque réseau institutionnel (WiFi scolaire filtré)


Un risque distinct a été anticipé : le device sera transporté dans l'établissement scolaire pour la démonstration, sur un réseau WiFi filtré (infrastructure Sophos identifiée). Ce type d'infrastructure pratique fréquemment de l'inspection profonde de paquets (DPI) susceptible de détecter et bloquer un tunnel OpenVPN par signature de protocole, indépendamment de l'état d'ouverture apparente d'un port.


## 7.1  Contournement envisagé : partage de connexion cellulaire


Une solution alternative a été proposée : utiliser le forfait de données mobiles du Redmi Note 14 (device secondaire, avec carte SIM active) en hotspot pour fournir une connexion internet indépendante au Redmi Note 10, contournant entièrement l'infrastructure réseau de l'établissement.

Dépendance identifiée. Le Redmi Note 10 agit comme exit-node Tailscale (rôle documenté dans le Portfolio principal) — Tailscale ne fournit pas la connexion internet de base, il l'utilise. La séquence de dépendance correcte est donc : connexion internet quelconque (hotspot) d'abord, puis stabilisation de Tailscale et attribution de son IP mesh stable (100.x.x.x), puis accessibilité SSH via cette IP — indépendamment du réseau physique sous-jacent.

Point de vigilance opérationnel. Un changement de réseau physique (WiFi maison vers hotspot cellulaire) impose un délai de reconnexion Tailscale (de l'ordre de 30 secondes à 2 minutes) pendant lequel le device reste temporairement injoignable par IP Tailscale. Une marge de stabilisation doit être prévue avant toute tentative de connexion SSH lors du changement de contexte réseau.

Recommandation retenue. Faire pointer le script Create-AnonNamespace.ps1 sur l'IP Tailscale stable du device (100.75.47.3 au moment de la rédaction) plutôt que sur une IP LAN locale (192.168.x.x ou 10.0.0.x), qui elle varie selon le réseau physique. Ce changement rend le script robuste à tout changement de réseau futur, pas seulement pour cette démonstration.


# 8. État actuel & items ouverts



## 8.1  État de conception


Cette session a été une session de conception et de discussion — aucune commande n'a été exécutée sur le device. L'ensemble des points ci-dessous reste à valider en conditions réelles.

Item

Statut

Suppression du chroot Kali

Effectuée

Disponibilité openvpn / dnsmasq (Termux)

Vérifiée par recherche — non testée sur device

Script Create-AnonNamespace.ps1 v2 (DNS + firewall)

Rédigé — non exécuté

Mini-firewall default-deny anon_ns

Conçu — non testé

Script pointant sur IP Tailscale plutôt que LAN

Recommandé — modification non appliquée

Test hotspot Note 14 → Note 10 → Tailscale

Non testé

Compte / config Proton VPN (fallback stable)

Non créé


## 8.2  Items ouverts pour la prochaine session


Exécution réelle : lancer le script v2 sur le device, valider l'installation des paquets, vérifier le comportement du firewall (test positif/négatif documenté en §5).

Test de connectivité DNS filtré : confirmer qu'une requête dnsmasq → 9.9.9.9 depuis anon_ns aboutit et que le trafic hors whitelist est bien bloqué.

Test du scénario hotspot : valider la reconnexion Tailscale et l'accessibilité SSH via IP Tailscale après bascule WiFi maison → hotspot cellulaire, avant le jour de la démonstration.

Préparation Proton VPN : obtenir une configuration .ovpn compatible avec le script (mêmes variables IP/port/protocole) pour le scénario de démonstration principal.

Test sur environnement de lab local (VM) : envisagé pour itérer plus rapidement sur les variantes de firewall et de DNS sans dépendre du device distant — piste évoquée pour une session future utilisant le lab Hyper-V existant.


# 9. Annexes



## Annexe A : Différence camellia / camellian


Clarifié en session : camellia est le codename de plateforme matérielle de base ; camellian désigne le variant régional Global/EEA sur cette même plateforme. Le device de ce projet est identifié en modèle commercial M2103K19G (Global), donc camellian. Certains outils bas niveau (bootloader, fastboot) peuvent afficher le codename de plateforme générique (camellia) plutôt que le variant précis — vérifier systématiquement le modèle exact via Paramètres > À propos ou getprop ro.product.model avant tout flash de firmware.


## Annexe B : Chipset — MediaTek Dimensity 700 (Helio)


Le device repose sur un chipset MediaTek, pas Qualcomm Snapdragon. Conséquence pratique documentée : l'outillage de flash diffère (SP Flash Tool / XiaoMiFlash pour MediaTek, contre fastboot/EDL QFIL pour Snapdragon), et le support communautaire de custom ROM est historiquement plus restreint côté MediaTek — confirmé par la recherche menée en session (peu de ROM natives disponibles pour camellia/camellian, principalement crDroid et CherishOS, complétées par des GSI génériques).

Pour l'usage de ce lab (réseau, conteneurisation, tests namespace), la différence de puissance CPU/GPU entre MediaTek et Snapdragon a été jugée non déterminante — les facteurs limitants réels restent la RAM disponible et le support kernel/root, indépendants du fabricant du chipset.


## Annexe C : Configuration mémoire virtuelle du device


Configuration swap hybride documentée en session, pour référence :

Composant

Taille

Priorité

RAM physique

~3,5-3,6 Go

—

zram

6 Go

100

swapfile (stockage)

8-9 Go

10

La priorité 100 du zram, très supérieure à la priorité 10 du swapfile, garantit que le kernel privilégie systématiquement la compression en RAM avant de recourir au stockage flash — limitant l'usure de la mémoire flash tout en conservant une marge de sécurité totale d'environ 17-18 Go de mémoire virtuelle pour absorber les pics de charge (namespace + OpenVPN + dnsmasq, potentiellement combinés à une future charge d'inférence IA locale).


## Annexe D : Custom ROM identifiées pour camellia/camellian


ROM

Type

Statut

crDroid

ROM native

Device officiellement supporté

CherishOS

ROM native

Confirmé fonctionnel (v5.3, Android 14)

Project Elixir

ROM native (famille Poco M3 Pro 5G)

Confirmé pour devices apparentés

LineageOS GSI

GSI (Generic System Image)

Compatible via Project Treble

Pixel Experience GSI

GSI

Compatible via Project Treble

Avertissement documenté. Un cas de brick rapporté dans la communauté XDA implique l'effacement de la partition boot après modification de la partition rescue — la recommandation retenue en session est de tester tout flash de ROM sur un device de rechange avant d'intervenir sur le Note 10 actuel, qui porte déjà trois sessions de configuration documentées.


# 10. Annotations personnelles


Cette session marque un changement de nature par rapport aux précédentes : pas de commande exécutée, pas de log à analyser, uniquement de la conception et de l'anticipation de risque. C'est pourtant l'une des sessions les plus utiles du projet — identifier qu'une démonstration académique pourrait échouer à cause de la mortalité d'un serveur VPN bénévole, avant que ça arrive, vaut largement plus que n'importe quel fix réactif en pleine présentation.

L'abandon du chroot Kali est également une décision de maturité technique plutôt qu'un renoncement. Le chroot ajoutait une couche entière de complexité (inventaire de paquets, PATH fix, transfert zero-nesting dédié) pour un objectif qui ne le justifiait pas — isoler un tunnel VPN ne demande pas un environnement offensif complet. Reconnaître qu'une architecture antérieure, bien que fonctionnelle, était sur-dimensionnée pour le besoin réel est une forme de rigueur aussi importante que celle de la faire fonctionner initialement.

Le constat sur AFWall+ et la portée des network namespaces (une app de filtrage Android ne voit pas un namespace qu'elle n'a pas créé) est un rappel utile : l'isolation réseau et la sécurité applicative sont deux couches distinctes, et une architecture qui isole bien peut simultanément introduire un angle mort de sécurité si cette distinction n'est pas explicitement traitée. Documenter ce risque et le mitiger par un firewall minimal directement dans le kernel, plutôt que de compter sur une app tierce, est cohérent avec la philosophie du projet depuis le début : comprendre la couche la plus basse plutôt que faire confiance à l'abstraction du dessus.

Enfin, la discussion élargie sur les prochaines étapes (Docker, Kubernetes, Terraform, custom ROM, IA locale conteneurisée) confirme que ce projet de home-lab n'est pas une fin en soi, mais une fondation délibérée. Les mêmes primitives kernel manipulées ce soir à la main — network namespaces, veth, iptables — sont exactement celles sur lesquelles reposent Docker et Kubernetes en production. Le choix de construire cette compréhension manuellement, sur un device contraint, avant de s'appuyer sur des outils qui l'abstraient, est une séquence d'apprentissage délibérée et cohérente.

Document confidentiel · Usage portfolio uniquement
