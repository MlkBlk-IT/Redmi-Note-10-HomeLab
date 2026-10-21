
# PRÉFACE DU PROJET



## Home-Lab Android & Infrastructure Mobile : Genèse et Vision


Auteur : Administrateur Infrastructure / Portfolio Technique

Appareil cible : Xiaomi Redmi Note 10 5G (Nom de code : camellia)

Période de réflexion et d'exécution : Août 2026

Transparence et Méthodologie de Documentation :

L'ensemble de la réflexion architecturale, des heures de diagnostic réseau, de l'ingénierie système bas niveau et des nuits passées à débugger le noyau Linux de ce téléphone a été réalisé à 100 % par mes soins. Afin de retranscrire ce travail complexe de la manière la plus claire et professionnelle possible, j'ai fait appel à l'Intelligence Artificielle. Elle a agi comme une assistante de rédaction, m'aidant à structurer mes notes brutes, à formater les documents et à mettre en page ce portfolio, me permettant ainsi de me concentrer sur l'essentiel : la technique.


## 1. L'Étincelle : Plus qu'un simple téléphone


L'idée derrière ce projet est née d'une question simple mais persistante : que se passe-t-il si l'on arrête de considérer un smartphone grand public comme un simple appareil de consommation, pour le traiter avec le même sérieux qu'un véritable serveur Linux bare-metal ?

Le Xiaomi Redmi Note 10 5G qui repose sur mon bureau n'est pas un monstre de puissance (3,5 Go de RAM, puce MediaTek). Mais sous son interface hyper-verrouillée (MIUI/HyperOS), il fait tourner un noyau Linux. Mon ambition n'était pas de simplement installer quelques applications pour faire du "bricolage", mais de descendre aux fondations du système. Je voulais repousser les limites de ce matériel contraint, le dépouiller de ses processus inutiles, et le forcer à héberger une infrastructure réseau digne d'un véritable Home-Lab.


## 2. La Philosophie du Projet : Contrôle et Indépendance


Aujourd'hui, l'industrie nous pousse vers des solutions Cloud faciles (clés en main) où l'on abandonne le contrôle de nos données et de notre routage en échange de la commodité. Ce projet est une démarche inverse.

Indépendance technique : Ne pas dépendre d'applications tierces (comme des proxys opaques ou des VPN commerciaux restrictifs), mais construire ma propre isolation réseau via les primitives natives du kernel (Network Namespaces, iptables).

Optimisation extrême : Apprendre à gérer la rareté. Faire tourner un serveur IA local (llama-server), un serveur de fichiers, et des tunnels chiffrés sur seulement 3,5 Go de RAM exige une compréhension intime de la mémoire (zRAM, swap, gestion du Doze mode).

Résilience : Transformer un appareil mobile, par nature instable (changement d'IP, perte de signal, mise en veille agressive), en un nœud d'infrastructure fiable et accessible à distance (Tailscale, Cloudflare).


## 3. Ce que vous allez lire dans ces documents


Les documents qui suivent cette préface ne sont pas de simples tutoriels. Ce sont des carnets de bord de mes sessions d'ingénierie. Vous y lirez mes réussites, mais aussi (et surtout) mes impasses. Vous verrez comment un simple problème de hotspot m'a conduit à auditer les conflits de ports DNS (Addendum III), ou comment l'exclusivité du VPN natif d'Android m'a poussé à contourner les filtres eBPF en déplaçant dynamiquement des interfaces réseau dans des environnements isolés (Addendums IV à VI).

Ce portfolio est la preuve concrète que les véritables limites de l'informatique ne sont pas toujours matérielles. Elles résident souvent dans les couches d'abstraction logicielles qu'on nous impose. Ce projet a été ma façon de les briser.

Bonne lecture.
