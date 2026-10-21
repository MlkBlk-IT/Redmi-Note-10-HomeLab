Home-Lab Android · Addendum V

ADDENDUM V

Diagnostic Network Namespace : eBPF, cgroup-BPF & Isolation ICMP

Suite aux Addendums I–IV, Home-Lab Android

Document de portfolio technique · Addendum V

Session — 14 août 2026

Device : Xiaomi Redmi Note 10 5G (camellia)

TABLE DES MATIÈRES


# 1. Résumé de la session


Cette session prolonge le diagnostic amorcé dans l'Addendum II (routage kernel et network namespaces) et l'Addendum III (VPN + firewall dans anon_ns). L'objectif initial était double : (1) obtenir un accès en lecture aux structures eBPF/cgroup-BPF du device, jusqu'ici bloqué par l'absence de bpftool, et (2) déterminer pourquoi le trafic sortant du network namespace anon_ns échoue de façon reproductible sur certains protocoles.

Un binaire bpftool arm64 statique a été localisé, transféré et validé fonctionnel sur le device malgré un kernel Android 13/HyperOS sans support CONFIG_DEBUG_INFO_BTF. L'inspection a permis d'identifier avec précision les programmes eBPF chargés par netd (cgroup_sock, cgroup_skb, xt_bpf allowlist/denylist), de dumper leurs maps de configuration, et d'éliminer plusieurs hypothèses par la preuve plutôt que par déduction.

Le symptôme initial (ping: sendmsg: Operation not permitted dans anon_ns) s'est révélé spécifique au protocole ICMP — confirmé par des tests différentiels sur UDP et TCP. Une hypothèse secondaire impliquant AdGuard Home et AdAway a été testée et formellement écartée par isolation de variables. Le test décisif de la session a porté sur OpenVPN (Proton VPN) : le tunnel monte instantanément hors du namespace, mais échoue systématiquement par timeout TLS à l'intérieur de anon_ns, sur tous les ports testés.

Conclusion de session : anon_ns bloque le trafic retour ou la négociation UDP bidirectionnelle pour au moins deux protocoles distincts (ICMP, OpenVPN/UDP), indépendamment d'AdGuard Home et d'AdAway. La cause exacte au niveau eBPF reste non confirmée — l'absence de BTF empêche bpftool cgroup tree/list/show de résoudre les points d'attachement réels, malgré la confirmation que les programmes concernés sont chargés et actifs dans le kernel.


# 2. Obtention et validation de bpftool



## 2.1  Constat de départ


Les sessions précédentes (Addendum II, III) avaient identifié l'absence de bpftool sur le device comme un blocage de diagnostic : impossible d'inspecter les programmes eBPF, les maps, ou les points d'attachement cgroup responsables du comportement réseau observé. Une première vérification en lecture seule (which bpftool, find /system /vendor) a confirmé son absence complète du système.


## 2.2  Recherche et sélection du binaire


Recherche d'un binaire arm64 pré-compilé pour Android, condition nécessaire pour éviter une compilation cross-toolchain complète sur le device. Le dépôt bnsmb/binaries-for-Android (GitHub) a été identifié comme source fiable, avec un binaire bpftool statique lié uniquement aux bibliothèques standard Android (libc, libm, libdl), testé sur Android 13 et versions ultérieures.

Deux tentatives de récupération ont été nécessaires :

URL directe bnsmb.de/files/public/Android/binaries_for_arm64/bpftool_static_5.16.0 → 404 Not Found (fichier déplacé ou renommé depuis la publication de la source XDA).

URL miroir GitHub github.com/bnsmb/binaries-for-Android/raw/refs/heads/main/binaries/bpftool_static_5.16.0 → succès, 1030 Ko transférés.


## 2.3  Validation


Le fichier récupéré a été vérifié avant exécution :

$ file /data/local/tmp/bpftool/data/local/tmp/bpftool: ELF executable, 64-bit LSB arm64, static,for Android 36, BuildID=4f17c1096780db2be06c09e68fb54a47, stripped

Le binaire est compilé pour Android API 36 (bien au-delà d'Android 13/HyperOS du device), mais son caractère statique lui a permis de s'exécuter sans erreur :

$ /data/local/tmp/bpftool versionbpftool v5.16.0-c446fdacb10dusing libbpf v1.4

📌 Pourquoi le mismatch de version n'a pas bloqué l'exécution. Un binaire statique n'embarque que des appels au kernel via syscalls (bpf(), notamment), pas de dépendance à des bibliothèques utilisateur versionnées. Le syscall bpf() est stable dans son ABI depuis son introduction ; un bpftool compilé pour un SDK Android plus récent reste fonctionnel sur un kernel plus ancien tant que les primitives BPF qu'il utilise sont supportées par ce kernel. C'est une différence importante entre compatibilité SDK (liée aux bibliothèques Android) et compatibilité kernel (liée aux syscalls) — la première a échoué à l'exécution (device Android 13, binaire ciblé Android 36) sans empêcher la seconde de fonctionner.


# 3. Cartographie des programmes eBPF actifs



## 3.1  Localisation du montage cgroup2


La première tentative d'inspection (bpftool cgroup tree, sans argument) a retourné une sortie vide. Investigation immédiate plutôt que conclusion prématurée : inspection des points de montage cgroup réels du système.

$ mount | grep cgroupnone on /dev/blkio type cgroup (...)none on /dev/cg2_bpf type cgroup2 (rw,nosuid,nodev,noexec,relatime)none on /dev/cpuctl type cgroup (...)none on /dev/cpuset type cgroup (...)none on /dev/memcg type cgroup (...)none on /dev/stune type cgroup (...)none on /sys/fs/cgroup type tmpfs (...)

Android n'utilise pas /sys/fs/cgroup comme racine cgroup2 — c'est un tmpfs organisant plusieurs hiérarchies cgroup v1 distinctes (blkio, cpuctl, cpuset, memcg, stune). Le seul point de montage cgroup2 réel, support natif d'eBPF, est /dev/cg2_bpf.


## 3.2  Inventaire des programmes chargés


bpftool prog list a révélé 21 programmes eBPF actifs dans le kernel, répartis en plusieurs catégories : 5 tracepoint (profiling CPU/GPU), 11 sched_cls (traffic control, non examinés cette session par manque de temps), 2 cgroup_skb, 4 socket_filter, 1 sched_act, et 1 cgroup_sock. Les trois derniers types sont directement pertinents pour le filtrage réseau par UID.

ID

Type

Pin path / rôle identifié

28

cgroup_sock

prog_netd_cgroupsock_inet_create — hook création socket AF_INET

21

cgroup_skb

Non pinné — probablement ingress (bpf_owner_match / traffic_account)

22

cgroup_skb

Non pinné — probablement egress (logique complète DOZABLE/STANDBY/POWERSAVE/RESTRICTED)

26

socket_filter

prog_netd_skfilter_allowlist_xtbpf — matché via iptables xt_bpf

27

socket_filter

prog_netd_skfilter_denylist_xtbpf — reject-with icmp-port-unreachable


## 3.3  Inventaire du filesystem BPF pinné


find /sys/fs/bpf a révélé une arborescence complète et non-triviale : programmes de profiling (time_in_state, gpu_work, gpu_mem), un sous-répertoire net_shared (CLAT — traduction IPv6/IPv4 pour le VPN 464XLAT), un sous-répertoire tethering (offload matériel du partage de connexion), et le sous-répertoire netd_shared contenant l'ensemble du firewall applicatif natif d'Android : cookie_tag_map, uid_owner_map, blocked_ports_map, data_saver_enabled_map, configuration_map, ainsi que les quatre programmes xt_bpf/schedact déjà identifiés.

📌 Pourquoi cette cartographie exhaustive était nécessaire avant toute conclusion. Le finding de l'Addendum II (Tailscale intercepte via ip rule à haute priorité) et celui de l'Addendum III (userspace-networking de Tailscale échappe à netfilter) concernaient des mécanismes visibles avec les outils standard (ip rule, iptables). Le blocage observé dans anon_ns ne suivait aucun de ces deux patterns connus, ce qui justifiait l'hypothèse d'un troisième mécanisme — eBPF cgroup-BPF — nécessitant un outil dédié pour investigation. Sans cette cartographie, l'hypothèse serait restée non vérifiable.


# 4. Élimination d'hypothèses par la preuve


Cette section documente trois hypothèses testées et écartées dans l'ordre chronologique de la session, chacune par lecture directe de configuration ou test différentiel — jamais par déduction seule.


## 4.1  Hypothèse : netd bloque activement via cgroup_sock/cgroup_skb


Le dump de map_netd_uid_owner_map (216 entrées) a confirmé la présence de l'UID Termux (10260) avec une valeur de règle rule = 0x81. Cependant, le dump de map_netd_configuration_map a révélé la clé UID_RULES_CONFIGURATION_KEY (valeur 1, confirmée par le code source AOSP consulté) avec enabledRules = 0.

Le code source netd (bpf_owner_match, platform/system/netd) contient la clause suivante :

BpfConfig enabledRules = getConfig(UID_RULES_CONFIGURATION_KEY);if (enabledRules) {    if ((enabledRules & DOZABLE_MATCH) && !(uidRules & DOZABLE_MATCH))        return BPF_DROP;    ...}

Avec enabledRules = 0, la condition if (enabledRules) est fausse et aucune règle DOZABLE/STANDBY/POWERSAVE/RESTRICTED n'est évaluée. Verdict : netd ne bloque aucun trafic par ce mécanisme au moment de l'inspection, quelle que soit la valeur de rule associée à un UID donné.


## 4.2  Hypothèse : filtres xt_bpf allowlist/denylist


Les deux règles iptables correspondantes ont été localisées :

0  0  RETURN  ...  match bpf pinned .../allowlist_xtbpf0  0  REJECT  ...  match bpf pinned .../denylist_xtbpf                    reject-with icmp-port-unreachable

Compteurs à zéro paquet/octet sur les deux règles. Un parcours exhaustif de la chaîne filter/OUTPUT (OUTPUT → oem_out → oem_cta_all → oem_data → oem_priou/oem_priot → bw_OUTPUT → bw_global_alert) a confirmé que le trafic transite par plusieurs sous-chaînes MediaTek/OEM custom mais ne rencontre jamais les règles xt_bpf dans ce chemin. Verdict : ces règles ne sont pas dans le chemin de sortie du trafic testé et n'expliquent pas le blocage.


## 4.3  Hypothèse : conflit AdGuard Home / AdAway


Test différentiel par isolation de variables, chaque combinaison testée avec le même appel UDP (nc -u vers 1.1.1.1:443 depuis anon_ns) :

Configuration testée

Résultat

AdGuard Home + AdAway actifs

Operation not permitted

Les deux désactivés

Succès (DONE, aucune erreur)

AdAway seul réactivé

Succès

AdGuard Home seul réactivé (AdAway coupé)

Succès

Les deux réactivés (état initial reproduit)

Succès — pas de blocage reproduit

Le blocage UDP ne s'est pas reproduit une fois les deux services réactivés dans le même ordre. Un nouveau test ICMP immédiatement après a confirmé que ping échouait toujours avec la même erreur, alors qu'un test UDP identique au précédent réussissait. Verdict : AdGuard Home et AdAway sont formellement écartés — le facteur commun aux échecs observés est le protocole (ICMP), pas l'état de ces deux services.

📌 Sur la valeur de la fausse piste. Consacrer du temps à tester puis écarter AdGuard Home/AdAway n'est pas du temps perdu : la première observation (blocage coïncidant avec les deux services actifs) était une corrélation plausible qui méritait vérification. La isolant de variables — un changement à la fois, un retour à l'état initial pour confirmer la non-reproduction — est ce qui a permis de l'écarter avec confiance plutôt que de la laisser comme doute non résolu dans la documentation finale.


# 5. Caractérisation du blocage réel : spécificité protocolaire



## 5.1  Tests différentiels par protocole


Une série de tests contrôlés, tous exécutés avec ip netns exec anon_ns depuis le même shell root, a permis d'isoler la portée exacte du blocage :

Protocole

Commande

Résultat

ICMP (raw socket)

ping -c1 -W2 1.1.1.1

sendmsg: Operation not permitted (reproductible à 100%)

UDP standard

nc -u 1.1.1.1 443

Aucune erreur — paquet émis sans blocage apparent

TCP standard

nc 1.1.1.1 443

Aucune erreur — pas de blocage visible

UDP bidirectionnel (OpenVPN)

openvpn --config ... (Proton, plusieurs ports)

TLS key negotiation failed — timeout 60s, systématique

Ce tableau révèle une distinction importante entre deux classes de test : les tests UDP/TCP simples avec nc n'attendent pas de réponse structurée du serveur distant et ne peuvent donc pas détecter un blocage du trafic retour. Le test OpenVPN, en revanche, nécessite un aller-retour complet (ClientHello TLS, réponse serveur, négociation de clé) — c'est le seul test UDP de la session à avoir réellement exercé le chemin bidirectionnel.


## 5.2  Confirmation hors-namespace : OpenVPN fonctionne nativement sur le device


Le processus OpenVPN déjà actif en arrière-plan (PID 10545, lancé lors d'une session antérieure) présentait dans ses logs une boucle d'échec constante :

TCP/UDP: Preserving recently used remote address: [AF_INET]37.19.199.154:1194TLS Error: TLS key negotiation failed to occur within 60 secondsTLS Error: TLS handshake failedSIGUSR1[soft,tls-error] received, process restarting

Ce comportement s'est répété sur cinq ports différents du même serveur Proton (80, 51820, 4569, 1194, 5060) sur plus d'une heure d'exécution continue — cohérent avec le comportement normal d'OpenVPN qui retente les ports de fallback listés dans le fichier .ovpn, mais confirmant qu'aucun de ces ports n'aboutit depuis anon_ns.

Test décisif : la même configuration, lancée hors du namespace directement sur l'hôte, a établi le tunnel en moins d'une seconde :

TUN/TAP device tun0 openedData Channel: cipher 'AES-256-GCM', peer-id: 65541Initialization Sequence Completed

📌 Pourquoi ce test tranche définitivement la question du serveur VPN. Avant ce test, l'hypothèse « le serveur Proton est indisponible ou le fichier .ovpn est obsolète » restait ouverte — cinq ports en échec sur le même serveur pouvait aussi bien indiquer un serveur mort qu'un blocage local. Le même fichier de configuration, sans aucune modification, négocié en moins d'une seconde hors du namespace, élimine cette hypothèse avec certitude : le serveur répond, la configuration est valide, la clé et le certificat sont corrects. Le seul delta entre les deux tests est la présence ou l'absence du network namespace anon_ns.


# 6. Bilan et items ouverts



## 6.1  Ce qui est confirmé


bpftool est maintenant fonctionnel sur le device (/data/local/tmp/bpftool), malgré l'absence de BTF, pour les opérations prog list, prog show, map dump, feature probe.

netd charge et maintient actifs des programmes cgroup_sock, cgroup_skb et xt_bpf, mais aucun n'est actuellement configuré pour bloquer du trafic (enabledRules = 0, compteurs xt_bpf à zéro).

Le blocage anon_ns touche au minimum ICMP (raw socket) et le trafic retour UDP bidirectionnel (OpenVPN) — pas un simple envoi UDP aveugle.

Le serveur Proton VPN et le fichier .ovpn sont valides et fonctionnels — confirmé par un handshake TLS réussi hors namespace en moins d'une seconde.

AdGuard Home et AdAway, seuls ou en combinaison, sont formellement écartés comme cause.


## 6.2  Ce qui reste non confirmé


Le point d'attachement réel des programmes cgroup_sock/cgroup_skb (IDs 28, 21, 22) — bpftool cgroup tree/list/show retournent vide à tous les niveaux testés (racine /dev/cg2_bpf, sous-cgroup uid_10260, sous-cgroup pid_11823), bloqué par l'absence de CONFIG_DEBUG_INFO_BTF sur ce kernel.

Les 11 programmes sched_cls (traffic control) n'ont pas été examinés cette session — piste ouverte pour une prochaine investigation.

Le mécanisme exact empêchant le trafic retour d'atteindre anon_ns (routage veth asymétrique, règle de retour manquante, ou autre mécanisme non encore identifié).


## 6.3  Voie pragmatique identifiée


Le test hors-namespace démontre qu'OpenVPN fonctionne immédiatement et sans obstacle sur ce device. Deux chemins concrets se dégagent pour la suite du projet :

Lancer OpenVPN hors du network namespace — solution immédiatement opérationnelle, au prix de l'isolation réseau que anon_ns était censé fournir.

Poursuivre le diagnostic du blocage anon_ns (routage veth, sched_cls, ou tentative de compilation d'un kernel avec BTF activé) — travail de session dédiée, cohérent avec la piste network namespace déjà documentée comme faisable mais non triviale dans l'Addendum II.

📌 Sur la valeur de cette session malgré l'absence de cause racine définitive. Contrairement aux sessions précédentes où l'abandon survenait faute d'outillage (bpftool absent), cette session a obtenu l'outillage manquant et l'a utilisé avec succès pour éliminer methodiquement plusieurs hypothèses plausibles. Ne pas trouver la cause exacte du blocage anon_ns n'invalide pas le travail : on sait maintenant, avec preuves à l'appui, ce qui n'est PAS la cause (netd actif, xt_bpf dans le chemin, AdGuard/AdAway), ce qui EST confirmé comme fonctionnel (le serveur VPN, la configuration, bpftool lui-même), et la portée précise du symptôme (protocolaire, pas général). C'est un état de diagnostic strictement meilleur qu'au début de la session, même sans résolution complète.


# 7. Annexes



## Annexe A : Commandes clés de la session


Récupération et validation de bpftool :

curl -L -o /data/local/tmp/bpftool \  https://github.com/bnsmb/binaries-for-Android/raw/refs/heads/main/binaries/bpftool_static_5.16.0chmod 755 /data/local/tmp/bpftool/data/local/tmp/bpftool version

Inspection des programmes et maps eBPF :

/data/local/tmp/bpftool prog list/data/local/tmp/bpftool prog show id <N> --pretty/data/local/tmp/bpftool map show pinned /sys/fs/bpf/netd_shared/<map>/data/local/tmp/bpftool map dump pinned /sys/fs/bpf/netd_shared/<map>/data/local/tmp/bpftool feature probe kernel

Recherche des points de montage et attachements cgroup :

mount | grep cgroupfind /dev/cg2_bpf -maxdepth 2 -type d/data/local/tmp/bpftool cgroup tree <path>/data/local/tmp/bpftool cgroup list <path>

Tests différentiels par protocole (depuis anon_ns) :

ip netns exec anon_ns ping -c1 -W2 1.1.1.1ip netns exec anon_ns timeout 3 nc -u 1.1.1.1 443 < /dev/nullip netns exec anon_ns timeout 3 nc 1.1.1.1 443 < /dev/null

Test OpenVPN hors namespace :

timeout 20 /data/data/com.termux/files/usr/bin/openvpn \  --config /data/data/com.termux/files/home/anon_ns/us-free-120.protonvpn.udp.ovpn \  --verb 3


## Annexe B : Structure UidOwnerValue (référence AOSP)


Confirmée par lecture du code source platform/system/netd (Git at Google) :

typedef struct {    uint32_t iif;   // Interface autorisée si IIF_MATCH actif    uint32_t rule;  // Bitmask de UidOwnerMatchType} UidOwnerValue;STRUCT_SIZE(UidOwnerValue, 2 * 4);  // 8 bytes#define UID_RULES_CONFIGURATION_KEY 1#define CURRENT_STATS_MAP_CONFIGURATION_KEY 2


## Annexe C : Inventaire /sys/fs/bpf/netd_shared (extrait pertinent)


prog_netd_cgroupsock_inet_createprog_netd_skfilter_denylist_xtbpfprog_netd_skfilter_allowlist_xtbpfprog_netd_schedact_ingress_accountprog_netd_skfilter_ingress_xtbpfprog_netd_skfilter_egress_xtbpfprog_netd_egress_statsprog_netd_ingress_statsmap_netd_uid_owner_mapmap_netd_configuration_mapmap_netd_blocked_ports_mapmap_netd_cookie_tag_map


# 8. Sources


bnsmb/binaries-for-Android — github.com/bnsmb/binaries-for-Android (binaire bpftool arm64 statique)

XDA Forums — « [GUIDE] How to compile bpftool for Android » (xdaforums.com/t/guide-how-to-compile-bpftool-for-android.4793469)

Android Open Source Project — platform/system/netd, bpf_progs/netd.c et bpf_shared.h (android.googlesource.com)

Android Open Source Project — « eBPF traffic monitoring » (source.android.com/docs/core/data/ebpf-traffic-monitor)

Android Open Source Project — « Extend the kernel with eBPF » (source.android.com/docs/core/architecture/kernel/bpf)

Fanju6/NetProxy-Magisk — github.com/Fanju6/NetProxy-Magisk (référence architecture cgroup eBPF alternative, non déployée cette session)

libbpf/bpftool — github.com/libbpf/bpftool (dépôt amont)


# 9. Annotations personnelles


Cette session marque un changement méthodologique important par rapport aux Addendums II et III : là où ces derniers documentaient des abandons faute d'outillage, celui-ci documente l'obtention de l'outillage manquant et son application rigoureuse. La différence n'est pas cosmétique. Avoir bpftool en main a transformé plusieurs « probablement » en « confirmé » ou « infirmé » — enabledRules = 0 n'est pas une supposition, c'est un octet lu directement dans une map noyau pinnée.

Le moment le plus instructif de la session a été la fausse piste AdGuard Home/AdAway. Une corrélation observée une seule fois (blocage coïncidant avec les deux services actifs) méritait d'être testée — et la seule façon de la vérifier proprement était l'isolation stricte des variables, une combinaison à la fois, avec retour à l'état initial pour confirmer la non-reproduction. Le résultat négatif n'est pas un échec : c'est exactement le genre de piste qui, non testée, aurait pu se retrouver citée comme cause probable dans une documentation moins rigoureuse.

Le test décisif reste celui d'OpenVPN hors namespace. Il illustre un principe qui traverse tout ce projet depuis l'Addendum II : la distinction entre « le service ne marche pas » et « le service ne marche pas dans ce contexte précis » est souvent la différence entre des heures de diagnostic mal dirigé et une conclusion en une commande. Le tunnel qui monte en moins d'une seconde hors du namespace, après plus d'une heure d'échecs répétés à l'intérieur, recentre immédiatement le problème sur anon_ns plutôt que sur VPNGate, Proton, ou la configuration OpenVPN elle-même.

Enfin, l'absence de CONFIG_DEBUG_INFO_BTF comme mur final de cette session illustre une limite réelle du diagnostic sur kernel mobile contraint : contrairement à un serveur Linux classique où recompiler le kernel avec BTF est une option raisonnable, sur un Redmi Note 10 5G rooté, cette voie impliquerait un kernel custom complet — hors scope, comme documenté pour l'overclock dans la session du 13 août. La bonne décision ici n'est pas de forcer cette voie, mais de basculer vers la solution pragmatique déjà validée : OpenVPN fonctionne, hors namespace, dès maintenant.
