# Disconnected OpenShift Agent-Based Installer Automation

Public-safe Ansible automation for a compact, three-node, bare-metal OpenShift 4.20.z installation using the Agent-based Installer.

The workflow runs from a bastion host. It renders installation artifacts, creates the Agent ISO, pauses for an engineer to mount the ISO through BMC virtual media, monitors the installation, and validates the finished cluster.

## What this repository does

- Validates the bastion, installer version, private input file paths, registry TLS, compact topology, and 9000 MTU requirement.
- Renders `install-config.yaml` and `agent-config.yaml` from Ansible variables.
- Reads the pull secret, SSH public key, and Artifactory CA from local files without printing their contents.
- Configures disconnected image mirrors through `imageContentSources` and `additionalTrustBundle`.
- Generates an `ImageDigestMirrorSet` installation manifest.
- Injects a master Chrony MachineConfig before Agent ISO generation.
- Generates `agent.x86_64.iso` and its SHA-256 checksum.
- Pauses for the manual BMC virtual-media mount and boot step.
- Monitors bootstrap and installation completion.
- Protects the generated kubeconfig and validates cluster health.

## What this repository does not do

- Mirror images into Artifactory.
- Configure Artifactory, DNS, switches, VLANs, load balancers, or BMC credentials.
- Mount virtual media or power-cycle bare-metal servers.
- Deploy post-install Operators or OpenStack services.
- Commit or display pull secrets, private keys, kubeconfig, certificates, private IP addresses, or customer data.

## Supported topology

- OpenShift 4.20.z only.
- Agent-based Installer.
- Disconnected registry with trusted Artifactory CA.
- Exactly three control-plane nodes and zero dedicated workers.
- End-to-end MTU 9000.
- Four NICs recorded per host; this base deployment configures the OCP machine/management interface only.

## Quick start

Read [docs/quickstart.md](docs/quickstart.md).

## Repository safety

Copy `examples/site-vars.example.yaml` outside the repository and populate it with approved site values. The `.gitignore` file excludes common secret files, generated artifacts, kubeconfig, installer state, and private inventory overrides.

Before committing, run:

~~~bash
git status --ignored
rg -n -i 'password|token|secret|kubeconfig|private key|BEGIN CERTIFICATE|registry\.' .
~~~

Review every match before pushing to a public repository.
