> ⚠️ **Note:** The rest of this repository (code, comments, scripts, docs, commits, issues) is written in **French**. Only this README is in English.

<div align="center">

<p align="center"><img src="https://raw.githubusercontent.com/MlkBlk-IT/Redmi-Note-10-HomeLab/main/docs/readme-header.svg" width="100%" alt="Redmi-Note-10-HomeLab visual header" /></p>

<img src="https://img.shields.io/badge/Linux%20kernel-00ff9c?style=for-the-badge&logo=linux&logoColor=00ff9c&labelColor=0d0d0d" /> <img src="https://img.shields.io/badge/eBPF-e10600?style=for-the-badge&labelColor=0d0d0d" /> <img src="https://img.shields.io/badge/network%20namespaces-e10600?style=for-the-badge&labelColor=0d0d0d" /> <img src="https://img.shields.io/badge/OpenVPN-00ff9c?style=for-the-badge&labelColor=0d0d0d" />

</div>

> Turning an Android smartphone (Xiaomi Redmi Note 10 5G / camellia) into a Linux server
> Kernel, network namespaces, eBPF, zram, exit-node VM

## Objective

Use a rooted Android phone as a 24/7 bare-metal Linux server: routing, network isolation, VPN tunnel, exit-node.

## Phases

| Phase | Description | Artifact |
|-------|-------------|----------|
| 0 | Preface & architecture | `00_Preface.md` |
| 1 | Kali + debloat + VPNGate | `Addendum_01_Kali_Debloat.md` |
| 2 | Kernel routing & namespaces | `Addendum_02_Routage_Kernel_Namespaces.md` |
| 3 | Security & firewall & DNS | `Addendum_03_DNS_Firewall_Risques.md` |
| 4 | Anonymous namespace (`anon_ns`) | `Addendum_04_Creation_Anon_Namespace.md` + PS1 script |
| 5 | eBPF & ICMP diagnostics | `Addendum_05_Diagnostic_eBPF_ICMP.md` |
| 6 | OpenVPN tunnel & switchover | `Addendum_06_Bascule_OpenVPN.md` |
| IV | Anonymous namespace (addendum) | `Addendum_IV_AnonNamespace.md` |
| VI | Exit-node VM (WinNAT/ICS bug) | `Addendum_VI_VM_ExitNode.md` |

## Documented WinNAT/ICS bug

Diagnosis of a native Windows hypervisor limitation:
- `WinNAT` (`New-NetNat`) and `ICS` tolerate **only one active NAT interface per host**
- Attempting parallel double-NAT → indeterminate state (`COM 0x80040201`)
- Sources: Microsoft Learn, system Q&A
- Pivot: dropped the Parrot VM → dedicated Windows Server RRAS

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
│   ├── 04_Phase_4_Anon_Namespace/  (+ .ps1 script)
│   ├── 05_Phase_5_Diagnostic_Avance/
│   └── 06_Phase_6_OpenVPN_Tunnel/
├── Addendum_IV_AnonNamespace.md
├── Addendum_VI_VM_ExitNode.md
└── CONTEXT_RESUME_WINSERVER_EXITNODE.md
```

## What this project demonstrates

1. **Low-level Linux kernel** — network namespaces (`ip netns`), veth, eBPF
2. **Diagnosing complex system bugs** — WinNAT/ICS single-NAT limitation
3. **Managing constrained resources** — zram, swapfile, 24/7 battery monitoring
4. **Resilience** — a clean architectural pivot when facing a non-negotiable limitation

## Author

Malik — IT Support student (DEP), self-taught
Project: August 13–16, 2026

---

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:00ff9c,50:03150c,100:e10600&height=100&section=footer" width="100%" alt="" />

**⚡ Built on CPU. Proven on disk. ⚡**

</div>
