ADDENDUM II

Routage Kernel, Conflit ip rule & Network NamespacesSuite aux documents Home-Lab Android

Document de portfolio technique · Addendum IISession — Août 2026Device : Xiaomi Redmi Note 10 5G (camellia)

TABLE DES MATIÈRES

1.  Résumé de la session  3

2.  Mise en place du tunnel OpenVPN  3

2.1  Source et méthode de transfert  3

2.2  Corrections de configuration  4

3.  Diagnostic : le tunnel monte, mais le trafic ne suit pas  4

3.1  Symptôme initial  4

3.2  Inspection des tables de routage  5

3.3  Analyse des règles ip rule  5

4.  Finding Technique : Tailscale gagne la course de priorité  6

5.  Piste identifiée : Network Namespaces  7

5.1  Principe  7

5.2  Vérifications kernel  7

5.3  Chemin d'implémentation  8

6.  État actuel & items ouverts  8

7.  Annexes  9

Annexe A : Commandes clés de diagnostic  9

Annexe B : Output ip rule show  10

Annexe C : Output ip route show table all (extrait)  10

8.  Sources  11

9.  Annotations personnelles  12


# 1. Résumé de la session


Cet addendum II documente une session de diagnostic approfondi sur le routage réseau du chroot Kali NetHunter. L'objectif était de faire sortir le trafic du chroot par un tunnel OpenVPN (VPNGate) plutôt que par l'exit node Tailscale déjà actif sur le même téléphone. Le tunnel OpenVPN a été monté avec succès, mais le trafic sortait systématiquement par la même IP publique, indépendamment du tunnel.Le diagnostic a révélé que Tailscale injecte des règles ip rule à haute priorité (13000–17000) qui interceptent le trafic avant consultation de la table main où vivent les routes OpenVPN. Le chroot Kali, partageant le même namespace réseau qu'Android, hérite de ce comportement.Une piste de résolution a été identifiée : l'utilisation de network namespaces Linux (CONFIG_NET_NS=y confirmé sur le kernel du device) pour isoler une stack réseau dédiée au chroot + OpenVPN, indépendante de Tailscale. Cette piste n'a pas été implémentée ce soir — décision de gestion de scope, cohérente avec la méthodologie des sessions précédentes.


# 2. Mise en place du tunnel OpenVPN



## 2.1  Source et méthode de transfert


La source de données est l'API publique VPNGate (http://www.vpngate.net/api/iphone/), qui expose un fichier CSV. Chaque ligne représente un serveur public ; la colonne 15 contient la configuration OpenVPN encodée en base64.La méthode de transfert fiable établie dans l'Addendum I (zero-nesting via base64) a été utilisée : téléchargement du CSV hors chroot (curl n'étant pas disponible dans le chroot Kali), extraction de la ligne du serveur ciblé, décodage base64, et écriture directe du fichier .ovpn dans le chroot via un one-liner SSH encodé.

Serveurs testés. public-vpn-186 (premier essai), puis public-vpn-161 (Japon, IP 219.100.37.122) — ce dernier a été retenu pour la session.


## 2.2  Corrections de configuration


--persist-key deprecated. Warning affiché au démarrage d'OpenVPN. Non bloquant — la clé est persistée correctement malgré le message. Ignoré.

--tmp-dir inexistant. OpenVPN héritait du répertoire temporaire de l'environnement Termux (/data/data/com.termux/files/usr/tmp), qui n'existe pas dans le chroot Kali. Fix explicite avec --tmp-dir /tmp dans la commande de lancement.

Résultat. Tunnel monté avec succès. Log OpenVPN confirme Initialization Sequence Completed. Interface tun0 créée dans le chroot. Routes 0.0.0.0/1 et 128.0.0.0/1 ajoutées vers tun0.


# 3. Diagnostic : le tunnel monte, mais le trafic ne suit pas



## 3.1  Symptôme initial


Peu importe que les commandes soient exécutées dans le chroot ou hors chroot, curl -4 ifconfig.me retourne systématiquement la même IP publique (66.130.89.244). Cette IP correspond à l'exit node Tailscale actif sur le device. Le tunnel OpenVPN, pourtant fonctionnel d'après ses logs, semble totalement ignoré par le trafic applicatif.

Première hypothèse écartée. Le problème ne vient pas d'une mauvaise configuration OpenVPN (routes absentes, DNS leak, ou interface non montée). ip route dans le chroot montre bien les routes 0.0.0.0/1 et 128.0.0.0/1 vers tun0. Le symptôme est plus profond.


## 3.2  Inspection des tables de routage


La commande ip route show table all a révélé une architecture de routage complexe sur le device. Plusieurs tables sont présentes simultanément :

Table

Contenu

Origine

main

Routes standard du système (LAN, default via WiFi, routes OpenVPN 0.0.0.0/1 et 128.0.0.0/1)

Kernel + OpenVPN

tun1

Routes associées à l'interface Tailscale (100.x.x.x/32, etc.)

Tailscale daemon

tun1_local

Routes locales spécifiques à Tailscale

Tailscale daemon

local

Routes locales du kernel

Kernel

Les routes OpenVPN vivent dans la table main, qui est la table par défaut. En conditions normales, elles devraient être consultées pour tout trafic non matché par des règles spécifiques.


## 3.3  Analyse des règles ip rule


L'inspection avec ip rule show a révélé l'origine réelle du problème. Tailscale injecte des règles de routage policy (RPDB) à des priorités très élevées :

Priorité 13000 : Règle associée à la table tun1. Match par plage d'UID (0–10186, 10188–10401, etc.), couvrant quasiment toutes les applications et processus userspace du système.

Priorité 16000 : Règle associée à la table tun1_local. Même mécanisme de matching par UID.

Priorité 17000 : Règle de fallback vers la table main — mais seulement pour les UIDs non matchés par les règles précédentes.

Mécanisme. Le kernel Linux consulte les règles ip rule par ordre de priorité croissante (plus le nombre est petit, plus la règle est prioritaire). Dès qu'une règle match (ici, par UID), le kernel route le paquet via la table associée (tun1 ou tun1_local) et ignore complètement les règles suivantes, y compris celle menant à la table main. Les routes OpenVPN, vivant dans main, ne sont donc jamais consultées.

Portée du problème. Ce n'est pas un conflit limité aux applications Android. Le chroot Kali partage le même namespace réseau que le kernel hôte (pas de isolation netns). Tout processus lancé dans le chroot hérite des mêmes règles ip rule et subit donc le même détournement de trafic vers Tailscale.


# 4. Finding Technique : Tailscale gagne la course de priorité


Ce finding formalise la découverte architecturale de la session. Il ne s'agit pas d'un échec de configuration OpenVPN mais d'une interaction systémique entre deux technologies réseau cohabitant sur un même kernel.

Tailscale contrôle le plan de contrôle réseau. L'application Tailscale Android ne se contente pas de créer une interface tun1. Elle manipule activement la RPDB (Routing Policy DataBase) du kernel via ip rule pour rediriger sélectivement le trafic des applications vers son tunnel. C'est un comportement légitime et documenté — Tailscale agit comme un VPN system-wide.

OpenVPN est « invisible » sans règles ip rule. OpenVPN, dans sa configuration par défaut, se contente d'ajouter des routes dans la table main. Il ne crée aucune règle ip rule. Par conséquent, il est systématiquement court-circuité par Tailscale qui a des règles à plus haute priorité.

Le matching par UID est global. Les plages d'UID couvertes par les règles Tailscale englobent quasiment tout le userspace (0–10401+). Il n'existe pas de « zone blanche » où OpenVPN pourrait opérer sans conflit dans le namespace par défaut.

Conséquence pour le chroot. Sans isolation réseau (network namespace), le chroot Kali est sujet aux mêmes règles que le reste du système. Le chroot isole le filesystem, pas la stack réseau.


# 5. Piste identifiée : Network Namespaces



## 5.1  Principe


Les network namespaces Linux sont le mécanisme standard d'isolation réseau au niveau kernel. Chaque namespace possède sa propre stack réseau complète : interfaces, tables de routage, règles ip rule, sockets, tables ARP, etc. — totalement indépendante des autres namespaces. C'est le même mécanisme utilisé par Docker, Kubernetes et les solutions enterprise de séparation perso/pro sur un même device.


## 5.2  Vérifications kernel


Trois vérifications en lecture seule ont été effectuées sur le device :

ip netns list : Résultat : vide. Aucun namespace personnalisé n'existe actuellement sur le device.

ip netns help : Résultat : binaire complet et fonctionnel. Les sous-commandes add, exec, delete, monitor, set, identify sont toutes disponibles.

zcat /proc/config.gz | grep NET_NS : Résultat : CONFIG_NET_NS=y. Le kernel Android du Redmi Note 10 5G supporte nativement les network namespaces. Aucun module externe ni recompilation n'est nécessaire.

Conclusion kernel. Rien ne bloque au niveau kernel. L'infrastructure est présente et fonctionnelle.


## 5.3  Chemin d'implémentation


La séquence théorique pour isoler le chroot Kali + OpenVPN serait :

Créer un nouveau namespace : ip netns add kali_vpn

Créer une paire veth (virtual ethernet) : ip link add veth0 type veth peer name veth1

Déplacer veth1 dans le namespace kali_vpn : ip link set veth1 netns kali_vpn

Configurer les adresses IP sur veth0 (hôte) et veth1 (namespace)

Activer le forwarding IP et le NAT sur l'hôte pour que le namespace puisse sortir via l'interface WiFi

Lancer OpenVPN à l'intérieur du namespace : ip netns exec kali_vpn openvpn --config ...

Lancer le chroot Kali à l'intérieur du même namespace : ip netns exec kali_vpn chroot /data/local/nhsystem/kali-arm64 ...

Le chroot hérite alors d'une stack réseau vierge, sans les règles Tailscale, où OpenVPN contrôle le routage par défaut.

Décision de scope. Ce chemin est identifié, documenté et validé comme faisable, mais n'a pas été implémenté ce soir. La configuration d'une paire veth + NAT + namespace sur Android rooté représente une session dédiée de plusieurs heures, avec des risques de coupure réseau si mal exécuté. Cohérent avec la méthodologie des sessions précédentes : documenter la piste, justifier l'abandon temporaire, et réaffecter l'énergie.


# 6. État actuel & items ouverts



## 6.1  État opérationnel


Tailscale : mesh fonctionnel (3 nœuds), exit-node actif, LAN Access réactivé, règles ip rule en place.

OpenVPN (VPNGate) : tunnel monté avec succès dans le chroot, interface tun0 créée, routes 0.0.0.0/1 et 128.0.0.0/1 présentes.

Routage : trafic du chroot toujours détourné vers Tailscale par les règles ip rule à priorité 13000–17000.

Kernel : CONFIG_NET_NS=y confirmé, ip netns fonctionnel, aucun namespace créé.

Debloat : 6 packages désactivés, ~1,1 GB RAM disponible.


## 6.2  Items ouverts


Implémentation network namespace : Création du namespace kali_vpn, paire veth, NAT, et exécution du chroot + OpenVPN à l'intérieur. Session dédiée requise.

Validation post-namespace : Confirmation que curl -4 ifconfig.me retourne l'IP du serveur VPNGate (219.100.37.122) et non celle de Tailscale.

Persistance du namespace au boot : Intégration de la création du namespace et du montage veth dans le script 50-kali-mount ou un script dédié.


# 7. Annexes



## Annexe A : Commandes clés de diagnostic


Vérifier routes :ip route

Vérifier toutes les tables :ip route show table all

Vérifier règles de routage :ip rule show

Vérifier interfaces :ip addr show

Vérifier support netns kernel :zcat /proc/config.gz | grep NET_NS

Lister namespaces :ip netns list

Aide netns :ip netns help

Lancer commande dans namespace :ip netns exec <name> <commande>

Test IP publique :curl -4 ifconfig.me

Lancer OpenVPN dans chroot (actuel) :chroot /data/local/nhsystem/kali-arm64 /usr/bin/env PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin openvpn --config /root/vpn161.ovpn --daemon --log /root/vpn161.log --tmp-dir /tmp


## Annexe B : Output ip rule show (schématique)


0:      from all lookup local13000:  from all uidrange 0-10186 lookup tun113000:  from all uidrange 10188-10401 lookup tun116000:  from all uidrange 0-10186 lookup tun1_local16000:  from all uidrange 10188-10401 lookup tun1_local17000:  from all lookup main...


## Annexe C : Output ip route show table all (extrait)


default via 10.0.0.1 dev wlan0 table main0.0.0.0/1 dev tun0 table main128.0.0.0/1 dev tun0 table main100.64.0.0/10 dev tun1 table tun1100.75.47.3 dev tun1 table tun1...


# 8. Sources


VPNGate API — http://www.vpngate.net/api/iphone/ (CSV public, colonne 15 = config OpenVPN base64)

Tailscale Documentation — « How Tailscale routes traffic » (tailscale.com/kb/1177/routing-traffic)

Linux man pages — ip-rule(8), ip-route(8), network_namespaces(7)

Android Kernel — CONFIG_NET_NS documentation (kernel.org/doc/Documentation/networking/netns.txt)

OpenVPN Reference Manual — openvpn.net/community-resources/reference-manual-for-openvpn-2-6/

Docker Networking — « Network namespaces and veth pairs » (docs.docker.com/network/)


# 9. Annotations personnelles


Cette session est un parfait exemple de diagnostic en profondeur : le symptôme apparent (« le VPN ne marche pas ») aurait pu mener à des heures de réinstallation d'OpenVPN, de changement de serveurs VPNGate, ou de bidouillage de configuration DNS. En remontant la pile réseau couche par couche — interface, routes, tables, règles — la vraie cause a été isolée en moins d'une heure : Tailscale possède un verrou de priorité sur la RPDB.La distinction entre « tunnel monté » et « trafic routé » est fondamentale. Beaucoup d'administrateurs confondent les deux. Un tunnel OpenVPN opérationnel (Initialization Sequence Completed, tun0 up, routes présentes) ne garantit absolument pas que le trafic emprunte ce tunnel. Les règles de routage policy (ip rule) ont la primauté absolue sur les routes statiques. C'est une leçon que je retiens pour toute infrastructure multi-VPN.La piste network namespace est la bonne voie. Elle est propre, standard, et kernel-native. Le fait que CONFIG_NET_NS=y soit déjà activé sur un téléphone Xiaomi de milieu de gamme montre à quel point le noyau Linux moderne est mature, même sur du matériel mobile. La prochaine session consacrée à cette piste devra gérer le NAT, le forwarding, et la persistance au boot — mais le fondement technique est solide.Encore une fois, abandonner l'implémentation ce soir n'est pas un échec. C'est une décision informée : la piste est validée, les risques sont identifiés (coupure réseau si mal configuré), et l'énergie est préservée pour une session dédiée. C'est exactement ce que j'attends de moi-même et de toute équipe avec laquelle je travaille : pas de bricolage, pas de workaround fragile, mais des solutions architecturalement correctes, documentées, et reproductibles.
