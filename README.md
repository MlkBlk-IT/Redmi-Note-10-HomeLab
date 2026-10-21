# Redmi Note 10 — HomeLab Android

> Transformation d'un smartphone Android (Xiaomi Redmi Note 10 5G / camellia) en serveur Linux
> Kernel, network namespaces, eBPF, zram, exit-node VM

## Objectif

Utiliser un téléphone Android rooté comme serveur bare-metal Linux 24/7 : routage, isolation réseau, tunnel VPN, exit-node.

## Phases

| Phase | Description | Artefact |
|-------|-------------|----------|
| 0 | Préface & architecture | `00_Preface.md` |
| 1 | Kali + debloat + VPNGate | `Addendum_01_Kali_Debloat.md` |
| 2 | Routage kernel & namespaces | `Addendum_02_Routage_Kernel_Namespaces.md` |
| 3 | Sécurité & firewall & DNS | `Addendum_03_DNS_Firewall_Risques.md` |
| 4 | Namespace anonyme (`anon_ns`) | `Addendum_04_Creation_Anon_Namespace.md` + script PS1 |
| 5 | Diagnostic eBPF & ICMP | `Addendum_05_Diagnostic_eBPF_ICMP.md` |
| 6 | OpenVPN tunnel & bascule | `Addendum_06_Bascule_OpenVPN.md` |
| IV | Namespace anonyme (addendum) | `Addendum_IV_AnonNamespace.md` |
| VI | VM Exit-Node (WinNAT/ICS bug) | `Addendum_VI_VM_ExitNode.md` |

## Bug WinNAT/ICS documenté

Diagnostic d'une limitation native de l'hyperviseur Windows :
- `WinNAT` (`New-NetNat`) et `ICS` ne tolèrent **qu'une seule interface NAT active par hôte**
- Tentative de double-NAT parallèle → état indéterminé (`COM 0x80040201`)
- Sources : Microsoft Learn, Q&A système
- Pivot : abandon Parrot VM → Windows Server RRAS dédié

## Structure

```
├── 00_Preface.md
├── 01_Projet_Mobile_Server/
├── 02_Diagnostic_Reseau/
├── 03_HomeLab_Android/
│   ├── 00_Document_Principal/
│   ├── 01_Phase_1_Kali_et_Debloat/
│   ├── 02_Phase_2_Routage_Kernel/
│   ├── 03_Phase_3_Securite_Firewall/
│   ├── 04_Phase_4_Anon_Namespace/  (+ script .ps1)
│   ├── 05_Phase_5_Diagnostic_Avance/
│   └── 06_Phase_6_OpenVPN_Tunnel/
├── Addendum_IV_AnonNamespace.md
├── Addendum_VI_VM_ExitNode.md
└── CONTEXT_RESUME_WINSERVER_EXITNODE.md
```

## Ce que ce projet démontre

1. **Kernel Linux bas niveau** — network namespaces (`ip netns`), veth, eBPF
2. **Diagnostic de bugs systèmes complexes** — WinNAT/ICS mono-NAT limitation
3. **Gestion de ressources contraintes** — zram, swapfile, monitoring batterie 24/7
4. **Résilience** — pivot architectural propre face à une limitation non contournable

## Auteur

Malik — étudiant DEP Soutien Informatique, autodidacte
Projet : 13-16 août 2026
