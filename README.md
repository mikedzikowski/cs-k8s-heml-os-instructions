# cs-k8s-heml-os-instructions

A simple markdown repo with instructions and resources on how to deploy the CrowdStrike security stack on K8s and OpenShift clusters

# CrowdStrike Falcon Installation Guide

[**Home**](./README.md) | [**IAR**](./iar.md) | [**Sensor**](./sensor.md) | [**KAC**](./kac.md)

## Overview

This guide provides installation instructions for CrowdStrike Falcon components for Kubernetes environments. Select the appropriate tab above for detailed instructions on installing and configuring each component.

### Components

- **Image Assessment & Response (IAR)**: Scans container images for vulnerabilities and provides remediation guidance.
- **Falcon Sensor**: Provides runtime protection for containers and Kubernetes nodes.
- **Kubernetes Admission Controller (KAC)**: Enforces security policies by scanning images before they are deployed.

## Prerequisites

- Kubernetes cluster with admin access
- CrowdStrike API credentials with appropriate permissions
- Helm 3.x installed

## Quick Start

1. Set up your environment variables:

   ```bash
   export FCSCLIENTID=abcdef1234567890abcdef1234567890
   export FSCSECRET=abcdef1234567890abcdef1234567890abcdef12
   export FCSCID=ABCDEF1234567890ABCDEF1234567890-12
   ```

2. Add the CrowdStrike Helm repository:

   ```bash
   helm repo add crowdstrike https://crowdstrike.github.io/falcon-helm
   helm repo update
   ```

3. Follow the component-specific installation instructions in the tabs above.

## Support

For assistance, please contact [CrowdStrike Support](https://support.crowdstrike.com/) or refer to our [documentation](https://docs.crowdstrike.com/).
