# Hardware

The OpenCUBE testbed is currently equipped with eight compute nodes and one login node. 

## Overview

| Node | CPU | Memory | HSN | Accelerators | OS |
|:-----|-----|--------|---------|--------------|----|
| cn01 | Ampere Altra | 512GB | HPE Slingshot | | OpenSUSE Leap 16 |
| cn02 | Ampere Altra | 512GB | HPE Slingshot | | OpenSUSE Leap 16 |
| cn03 | Ampere Altra | 512GB | HPE Slingshot | 1x Xilinx U55C | OpenSUSE Leap 16 |
| cn04 | Ampere Altra | 512GB | HPE Slingshot | 1x Xilinx U55C | Talos v1.13.4 |
| cn05 | AmpereOne | 512GB | HPE Slingshot | 1x Xilinx U55C | OpenSUSE Leap 16 |
| cn06 | AmpereOne | 512GB | HPE Slingshot | 1x Xilinx U55C | OpenSUSE Leap 16 |
| cn07 | AmpereOne | 512GB | HPE Slingshot | | OpenSUSE Leap 16 |
| cn08 | AmpereOne | 512GB | HPE Slingshot | | OpenSUSE Leap 16 |



## Network

Each node is equipped with two network types:

1. Client Access Network
2. High-Speed Network

### Client Access Network

The Client Access Network uses two 10Gb/s Ethernet links and uses regular TCP/IP-based communication. 

### High-Speed Network

The High-Speed network is based on HPE Slingshot, which provides one 200Gb/s link. The Slingshot NIC Cassini 11 (CXI) exposes both a regular, emulated TCP/IP-based port, as well as an RDMA character device `/dev/cxi*`. Use a CXI-enabled libfabric to use Slingshot-provided RDMA communication. See [Slingshot](../slingshot/slingshot) for more information.

## Accelerators

Five Xilinx U55C FPGAs are installed in the testbed. Their location is provided in the [Overview](#overview). Programming is possible via out-of-band USB management, which are connected to the OpenCUBE login node. See [FPGA](../fpga) for more information.

## Operating System

The final OpenCUBE deployment will use the Talos Operating System. Currently, only `cn04` uses Talos. The remaining nodes use an OpenSUSE Leap 16. 

Note that raw SSH login is only possible to the OpenSUSE Leap systems. For system access to the final, Talos-based deployment, refer to [System Access](../system-access/overview).