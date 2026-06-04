# Azure Virtual Desktop (AVD) POC Deployment

This repository contains a step-by-step guide to deploy an Azure Virtual Desktop Proof of Concept environment.

## What's Included

| File | Description |
|------|-------------|
| [AVD-POC-Simple-Guide.md](AVD-POC-Simple-Guide.md) | Complete deployment guide — follow steps 1 through 23 |
| [image-template-fixed.json](image-template-fixed.json) | Azure Image Builder JSON template with generic placeholders |
| [aib-template.json](aib-template.json) | ARM template for Image Builder (alternative approach) |

## Getting Started

1. Open **[AVD-POC-Simple-Guide.md](AVD-POC-Simple-Guide.md)** and follow the steps sequentially.
2. The guide covers:
   - Resource Group, VNet, NSG, and NAT Gateway setup
   - VNet peering to an existing AD domain controller
   - Host Pool, Workspace, and Application Group creation
   - Custom golden image build (FSLogix, Teams, Adobe Reader)
   - Session Host deployment (domain-joined)
   - FSLogix profile storage with Azure Files + Private Endpoint
   - NTFS permissions and user assignment
   - End-to-end validation

## Prerequisites

- An Azure subscription with Owner or Contributor access
- Azure CLI (`az`) installed — [Install guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- An existing Active Directory Domain Controller (already deployed in Azure or on-premises with connectivity)
- Basic familiarity with Azure networking and Active Directory

## Custom Image (Optional)

If you want a pre-configured golden image with FSLogix, Teams, and Adobe Reader:
- Follow **Steps 10a and 10b** in the guide
- Use `image-template-fixed.json` — replace the placeholders with your values using the `sed` commands provided in the guide

If you prefer a marketplace image, skip Steps 10a/10b and use Step 11 Option B.

## License

This project is provided as-is for POC/lab purposes.
