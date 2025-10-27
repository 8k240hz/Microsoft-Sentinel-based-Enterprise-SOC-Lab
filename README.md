
├──.gitignore
├── LICENSE
├── README.md
├── arm-templates/
│   ├── network.json
│   └── network.parameters.json
├── images/
│   ├── architecture-network.png
│   └── dashboard-screenshot.png
├── kql-queries/
│   ├── 01-Credential-Access/
│   │   ├── T1110.001-Linux-SSH-Brute-Force.kql
│   │   ├── T1110.001-Windows-RDP-Brute-Force.kql
│   │   └── T1110.003-Password-Spraying.kql
│   ├── 02-Privilege-Escalation/
│   │   ├── T1078-User-Added-To-Privileged-Group.kql
│   │   ├── T1134-Special-Privileges-Assigned.kql
│   │   └── T1543.003-Suspicious-New-Service.kql
│   ├── 03-Lateral-Movement/
│   │   ├── T1021.001-Cross-Server-RDP-Connection.kql
│   │   ├── T1021.002-Access-To-Admin-Shares.kql
│   │   └── T1046-Internal-Port-Scanning.kql
│   └── 04-Execution/
│       ├── T1059.001-PowerShell-Download-Cradle.kql
│       ├── T1059.001-PowerShell-Encoded-Command.kql
│       └── T1218-LOLBins-Execution.kql
└── workbook-templates/
    └── soc-monitoring-dashboard.json
```

-----

### File Content

Below is the content for each of the essential files in the structure.

#### `README.md` (Main Project File)

# Enterprise SOC Lab with Microsoft Sentinel (Azure Free Tier)

This repository contains the architecture, deployment guide, and technical assets for building a fully functional, enterprise-style Security Operations Center (SOC) lab using Microsoft Sentinel on the Azure free tier. This project is designed to provide hands-on experience with a cloud-native SIEM at zero initial cost.

## Overview

The goal of this project is to construct a high-fidelity security monitoring environment that simulates a real-world enterprise network. By leveraging the Azure free account and the Microsoft Sentinel free trial, you can gain practical skills in cloud security, threat detection, and incident response without financial investment. A core learning objective is Security FinOps—the discipline of managing cloud costs within a security context.

## Features

  - **Centralized Log Collection:** Ingests security telemetry from a diverse environment, including:
      - 5 Windows Server 2022 VMs
      - 3 Ubuntu Server 22.04 VMs
      - 1 pfSense Community Edition firewall for network traffic logging
  - **Advanced Threat Detection:** Includes 15+ custom KQL queries designed to detect specific adversary tactics and techniques mapped to the MITRE ATT\&CK® framework.
  - **Interactive SOC Dashboard:** A custom Microsoft Sentinel Workbook provides at-a-glance operational visibility into key security metrics.
  - **Infrastructure as Code:** Includes ARM templates to automate the deployment of the core network infrastructure. [1]

## Architecture

This lab implements a standard hub-and-spoke network topology to ensure all traffic is inspected and logged.

### Network Topology

\!(./images/architecture-network.png)

The architecture consists of a single Azure Virtual Network (`vnet-soc-lab`) segmented into three subnets. A pfSense firewall, deployed as a Network Virtual Appliance (NVA), acts as the central security hub. A User-Defined Route (UDR) is applied to the endpoint subnet, forcing all outbound traffic through the pfSense NVA for inspection.

## Deployment Guide

This is a high-level overview of the deployment process.

### Step 1: Azure Free Account & Cost Management

1.  Sign up for a new [Azure free account](https://azure.microsoft.com/en-us/free/).
2.  **CRITICAL:** Navigate to **Cost Management + Billing** in the Azure portal. Create a budget (e.g., $1) and configure spending alerts to be notified via email before you exceed the free limits.
3.  Remember to **Stop (deallocate)** virtual machines from the Azure portal when not in use to stop compute charges.

### Step 2: Deploy Network Infrastructure

1.  Clone this repository to your local machine.
2.  Use the provided ARM templates in the `/arm-templates` directory to deploy the virtual network and subnets. This automates the creation of the network backbone for the lab. [2]bash
    az deployment group create --resource-group rg-soc-lab --template-file./arm-templates/network.json --parameters./arm-templates/network.parameters.json
    ```
    
    ```

### Step 3: Deploy pfSense Firewall

1.  Follow a guide to prepare a pfSense Community Edition image locally using Hyper-V or another hypervisor. [3]
2.  Ensure the virtual disk is a **fixed-size VHD** (not VHDX) and the VM is **Generation 1**. [3]
3.  Install the Azure Linux Agent (`waagent`) on the pfSense VM before deprovisioning and shutting it down. [4, 5]
4.  Upload the VHD to an Azure Storage Account and create a Managed Image from it.
5.  Deploy a B1s VM from this custom image, attaching two network interfaces (WAN and LAN) and enabling IP forwarding on the LAN interface.

### Step 4: Deploy Endpoints

1.  Deploy five **Windows Server 2022** VMs and three **Ubuntu Server 22.04** VMs.
2.  **CRITICAL:** Ensure you select the **Standard\_B1s** size for all VMs to stay within the 750-hour/month free allowance.
3.  Place all eight endpoint VMs in the `snet-endpoints` subnet. Do not assign public IP addresses.

### Step 5: Configure Sentinel & Log Ingestion

1.  Create a new Log Analytics Workspace and onboard Microsoft Sentinel to activate the 31-day free trial.
2.  Use the **Windows Security Events (AMA)** and **Syslog (AMA)** data connectors to create Data Collection Rules (DCRs).
3.  For the Windows DCR, use a **Custom** configuration with a specific XPath query to collect only the event IDs needed for the detection rules. This is essential for managing data volume. [6]
4.  Configure pfSense to send firewall logs to the private IP of one of the Linux VMs (the syslog collector). [7]
5.  Create a DCR for **Custom Logs (AMA)** to collect the pfSense logs from the collector VM's syslog file.

## Detection Rules Catalog

The KQL queries for the analytics rules are located in the [`/kql-queries`](./kql-queries) directory, organized by MITRE ATT\&CK tactic.

| Rule Name | MITRE ATT\&CK Tactic/Technique | Link to Query |
| :--------------------------------------------------------------------- | :---------------------------- | :-------------------------------------------------------------------------------------------------------- |
|(./kql-queries/01-Credential-Access/T1110.001-Windows-RDP-Brute-Force.kql) | TA0006 / T1110.001 | `T1110.001-Windows-RDP-Brute-Force.kql` |
|(./kql-queries/01-Credential-Access/T1110.001-Linux-SSH-Brute-Force.kql) | TA0006 / T1110.001 | `T1110.001-Linux-SSH-Brute-Force.kql` |
|(./kql-queries/01-Credential-Access/T1110.003-Password-Spraying.kql) | TA0006 / T1110.003 | `T1110.003-Password-Spraying.kql` |
| [User Added to Privileged Group](./kql-queries/02-Privilege-Escalation/T1078-User-Added-To-Privileged-Group.kql) | TA0004 / T1078 | `T1078-User-Added-To-Privileged-Group.kql` |
|(./kql-queries/02-Privilege-Escalation/T1543.003-Suspicious-New-Service.kql) | TA0004 / T1543.003 | `T1543.003-Suspicious-New-Service.kql` |
|(./kql-queries/03-Lateral-Movement/T1021.002-Access-To-Admin-Shares.kql) | TA0008 / T1021.002 | `T1021.002-Access-To-Admin-Shares.kql` |
|(./kql-queries/03-Lateral-Movement/T1046-Internal-Port-Scanning.kql) | TA0007 / T1046 | `T1046-Internal-Port-Scanning.kql` |
|(./kql-queries/04-Execution/T1059.001-PowerShell-Encoded-Command.kql) | TA0002 / T1059.001 | `T1059.001-PowerShell-Encoded-Command.kql` |
|(./kql-queries/04-Execution/T1059.001-PowerShell-Download-Cradle.kql) | TA0002 / T1059.001 | `T1059.001-PowerShell-Download-Cradle.kql` |

## SOC Monitoring Dashboard

The SOC monitoring dashboard is an interactive Microsoft Sentinel Workbook. The ARM template for this workbook can be found in [`/workbook-templates/soc-monitoring-dashboard.json`](./workbook-templates/soc-monitoring-dashboard.json). You can deploy this by navigating to **Microsoft Sentinel \> Workbooks**, selecting **Add workbook**, and then using the "Advanced Editor" (`</>`) to paste the JSON content from the file.

\!(./images/dashboard-screenshot.png)

## Disclaimer

**This project relies on Azure's free offerings, which have strict limits. Exceeding these limits will result in real-world costs.**

  - **Monitor Costs:** Regularly check the **Cost Management + Billing** page in the Azure portal.
  - **Set Alerts:** Configure budget alerts immediately after creating your account.
  - **Deallocate Resources:** Always **Stop (deallocate)** VMs when you are finished with a lab session.
  - **Deprovision:** Tear down all resources before your Azure credits or the Sentinel trial expires to avoid unexpected bills.

<!-- end list -->

````

---

#### `/kql-queries/01-Credential-Access/T1110.001-Windows-RDP-Brute-Force.kql`

```kql
// MITRE ATT&CK Technique: T1110.001 - Brute Force: Password Guessing
// Tactic: Credential Access
// Description: Detects a high number of failed RDP logon attempts (Logon Type 3) from a single source IP to a single host.
// Data Source: SecurityEvent (Windows)

let threshold = 10;
let timeWindow = 15m;
SecurityEvent

| where TimeGenerated > ago(timeWindow)
| where EventID == 4625 and LogonType == 3
| where SubStatus in ("0xc000006a", "0xc0000064") // Bad password or unknown user
| summarize FailedAttempts = count() by IpAddress, Computer, TargetAccount
| where FailedAttempts > threshold
````

-----

#### `/workbook-templates/soc-monitoring-dashboard.json`

```json
{
  "version": "Notebook/1.0",
  "items":,
  "$schema": "https://github.com/Microsoft/Application-Insights-Workbooks/blob/master/schema/workbook.json"
}
```

-----

#### `/arm-templates/network.json`

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vnetName": {
            "type": "string",
            "defaultValue": "vnet-soc-lab",
            "metadata": {
                "description": "Name of the Virtual Network."
            }
        },
        "vnetAddressPrefix": {
            "type": "string",
            "defaultValue": "10.10.0.0/16",
            "metadata": {
                "description": "Address prefix for the Virtual Network."
            }
        },
        "wanSubnetName": {
            "type": "string",
            "defaultValue": "snet-pfsense-wan",
            "metadata": {
                "description": "Name of the WAN subnet."
            }
        },
        "wanSubnetPrefix": {
            "type": "string",
            "defaultValue": "10.10.1.0/24",
            "metadata": {
                "description": "Address prefix for the WAN subnet."
            }
        },
        "lanSubnetName": {
            "type": "string",
            "defaultValue": "snet-pfsense-lan",
            "metadata": {
                "description": "Name of the LAN subnet."
            }
        },
        "lanSubnetPrefix": {
            "type": "string",
            "defaultValue": "10.10.2.0/24",
            "metadata": {
                "description": "Address prefix for the LAN subnet."
            }
        },
        "endpointsSubnetName": {
            "type": "string",
            "defaultValue": "snet-endpoints",
            "metadata": {
                "description": "Name of the Endpoints subnet."
            }
        },
        "endpointsSubnetPrefix": {
            "type": "string",
            "defaultValue": "10.10.3.0/24",
            "metadata": {
                "description": "Address prefix for the Endpoints subnet."
            }
        }
    },
    "resources": [
        {
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2020-11-01",
            "name": "[parameters('vnetName')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "[parameters('vnetAddressPrefix')]"
                    ]
                },
                "subnets":",
                        "properties": {
                            "addressPrefix": ""
                        }
                    },
                    {
                        "name": "",
                        "properties": {
                            "addressPrefix": ""
                        }
                    },
                    {
                        "name": "",
                        "properties": {
                            "addressPrefix": ""
                        }
                    }
                ]
            }
        }
    ]
}
```

-----

#### `/arm-templates/network.parameters.json`

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vnetName": {
            "value": "vnet-soc-lab"
        }
    }
}
```

-----

#### `LICENSE`

```
MIT License

Copyright (c) 2024

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
