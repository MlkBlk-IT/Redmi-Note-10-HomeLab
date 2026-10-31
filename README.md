# Redmi Note 10 HomeLab

A documented experiment that turns a rooted Xiaomi Redmi Note 10 5G into a small Linux server.

## Objective

The project explores routing, network isolation, VPN connectivity, eBPF diagnostics, zram and an exit-node design on Android hardware.

## Project phases

| Phase | Focus |
| --- | --- |
| 0 | Foundations and architecture |
| 1 | Kali, debloating and VPNGate |
| 2 | Kernel routing and network namespaces |
| 3 | DNS, firewall and security risks |
| 4 | An isolated namespace |
| 5 | eBPF and ICMP diagnostics |
| 6 | OpenVPN tunnel and switchover |
| VI | Exit-node VM and the WinNAT/ICS limitation |

The detailed procedure and evidence are kept in the phase documents in this repository.

## Important network finding

Windows WinNAT and ICS were not reliable for the planned parallel double-NAT design. The project documents the limitation and the resulting pivot toward a dedicated Windows Server RRAS design.

## Safety and scope

This is a personal lab. Test only on hardware and networks you control. Keep root, VPN and routing changes reversible and document the current state before applying them.

## Status

Documentation and lab work in progress. Hardware, Android version and kernel support affect the result.

## Author

Built by [Malik Belkacemi](https://github.com/MlkBlk-IT).
