CONTEXT_RESUME_WINSERVER_EXITNODE — 2026-08-15 (soir)

## CHANGEMENT DE PLAN MAJEUR CE SOIR
Le plan initial (Parrot Security = exit-node) est ABANDONNÉ pour la partie NAT/exit-node.
Parrot n'a plus AUCUN rôle dans ce projet.

## NOUVEAU PLAN CONFIRMÉ
Une seule nouvelle VM **Windows Server 2022 (ou 2025)**, nom prévu "WinServer-ExitNode", qui fait TOUT elle-même :
1. RRAS (NAT) — pour son propre accès Internet, sur son propre switch/sous-réseau, INDÉPENDANT de WinSrv2025
2. Tailscale — en exit-node du tailnet
3. Client OpenVPN — pour masquer/sortir le trafic

## ARCHITECTURE RÉSEAU DÉCIDÉE
- Nouvelle VM branchée sur **vSwitch_EXIT** (192.168.90.0/24) — PAS vSwitch_WAN
- Raison explicite de l'utilisateur : doit être invisible/séparée des VMs sous WinServer2025 (qui sont sur vSwitch_WAN, 192.168.137.0/24)
- Cette VM installe RRAS et fait son propre NAT sur vSwitch_EXIT — indépendant de WinSrv2025_NAT
- PAS de conflit attendu car deux switches différents = deux instances RRAS/NAT indépendantes (contrairement à 2 WinNAT sur le MÊME switch qui cassent, cf. bug de ce soir)
- Reste à déterminer : comment cette VM va elle-même sortir vers Internet (2e carte réseau ? Comment WinSrv2025 sort-elle déjà vers Internet — ICS ? Default Switch ? à vérifier avant de configurer RRAS dessus, pour éviter de re-déclencher le conflit "une seule cible ICS par hôte")

## CAUSE RACINE DU BUG DE CE SOIR (résolu, pour référence)
Le setup initial (EXIT_NAT via `New-NetNat` sur vSwitch_EXIT, IP statique sur Parrot) ne fonctionnait jamais malgré ARP résolu, IP correcte, routage correct, firewall clean, MAC correcte après fix cloned-mac-address.
CAUSE CONFIRMÉE (doc Microsoft officielle, learn.microsoft.com, mise à jour 14 août 2025) :
> "Currently, you're limited to one NAT network per host" — WinNAT ne supporte qu'UN SEUL NAT actif par hôte. Créer EXIT_NAT en parallèle de WinSrv2025_NAT (déjà actif via RRAS dans la VM WinSrv2025) place le système "into an unknown state" — le 2e NAT existe dans la config mais n'est pas réellement fonctionnel au niveau moteur.
Confirmé aussi : ICS (Internet Connection Sharing) est limité à UNE SEULE cible privée à la fois par hôte (confirmé Wikipedia + doc technique officielle) — tentative d'EnableSharing(1) sur une 2e cible échoue avec erreur COM 0x80040201, documentée comme bug/limite connue depuis des années (GitHub issues multiples), pas de fix Microsoft.
Le Default Switch a le MÊME problème (utilise ICS en interne) — confirmé par un Q&A Microsoft du 16 février 2026 : "you cannot have both the automated Default Switch and manual ICS active simultaneously."

## SOLUTION TEMPORAIRE TESTÉE CE SOIR (abandonnée, gardée pour référence)
Parrot rebranchée sur vSwitch_WAN (même switch que WinSrv2025), IP statique 192.168.137.50/24, gateway 192.168.137.1 → INTERNET CONFIRMÉ FONCTIONNEL (ping 8.8.8.8 OK). Mais dépendance totale à WinSrv2025 allumée — rejetée par l'utilisateur (contrainte RAM + veut séparation stricte entre les 2 projets/labs).

## INFRA HÔTE (rappel, inchangé)
Dell Latitude 5420, Win11 Pro, 32GB RAM, PowerShell admin, IP statique côté hôte.
Internet de l'hôte = WiFi uniquement (pas Ethernet).
vSwitch_LAN / vSwitch_WAN (Internal, 192.168.137.0/24, NAT via RRAS dans VM WinSrv2025) / vSwitch_EXIT (Internal, 192.168.90.0/24, actuellement sans NAT fonctionnel) / Default Switch (existe, non utilisé activement).
VMs Hyper-V existantes visibles : ExitNode (ex-Parrot, Running), Parrot (Off, doublon probable), Ubuntu (Off), Win11 (Off), Winserver2022 (Off — PROBABLEMENT LA VM À UTILISER POUR LE NOUVEAU PLAN), WinSrv2025 (Running).

## RÈGLES DE TRAVAIL DE L'UTILISATEUR (CRITIQUES, inchangées)
Malik, québécois, communique direct/cash, déteste se faire deviner des solutions sans vérification web préalable.
RÈGLE ABSOLUE : TOUJOURS chercher/vérifier sur le web (Microsoft, GitHub, sources officielles) AVANT de proposer une commande ou un diagnostic — ne JAMAIS improviser en premier.
Une commande à la fois, attendre confirmation du résultat avant de continuer.
Ne pas halluciner l'état du système — toujours demander le résultat réel de commande plutôt que supposer.
Ne pas re-proposer une option déjà explicitement rejetée (ex: dépendance à WinSrv2025) sans nouvelle info qui change la donne.

## PROCHAINE ÉTAPE IMMÉDIATE
Vérifier comment WinSrv2025 obtient actuellement son accès Internet (quelle carte réseau, quel mécanisme — ICS ? carte dédiée ?) pour répliquer un modèle équivalent et fonctionnel sur la nouvelle VM WinServer-ExitNode, sans redéclencher un conflit ICS/NAT au niveau de l'hôte Hyper-V.
Ensuite : installer RRAS sur WinServer-ExitNode (rôle Remote Access, config Custom → NAT, interface vSwitch_EXIT en Private, interface sortante en Public/NAT).
Puis : Tailscale (exit-node) et client OpenVPN sur cette même VM.

## AUTRES PROJETS ACTIFS (hors scope, mentionnés mais mis de côté)
OpenClaw V2, Hermes Agent, LM Studio — tous sur le Dell, hors scope pour ce chantier.
