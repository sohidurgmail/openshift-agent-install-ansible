# Quick start: disconnected Agent-based OpenShift installation

## Scope

This repository automates a compact, three-node OpenShift 4.20.z cluster installation from a bastion host. It is designed for a disconnected environment with an already prepared Artifactory mirror registry.

The BMC virtual-media mount and bare-metal boot are intentionally manual steps.

## 1. Prepare a private site variables file

Copy the example outside the repository:

~~~bash
cp examples/site-vars.example.yaml ~/site-vars.yaml
chmod 600 ~/site-vars.yaml
~~~

Populate this private file with approved values for:

- Cluster name and DNS base domain.
- Installation directory.
- Mirror registry hostname and CA file.
- Pull-secret file.
- SSH public-key file.
- Machine, cluster, and service networks.
- API and ingress VIPs.
- DNS and NTP servers.
- Three hostnames, IPs, MAC addresses, interface names, and root-disk WWNs.
- Mirror mappings supplied by the registry/image-mirroring team.
- A unique deployment ID for every installation attempt.

Do not commit `~/site-vars.yaml`.

## 2. Validate syntax

~~~bash
ansible-playbook -i inventories/site/hosts.yaml playbooks/preflight.yaml --syntax-check
ansible-playbook -i inventories/site/hosts.yaml playbooks/prepare_install.yaml --syntax-check
ansible-playbook -i inventories/site/hosts.yaml playbooks/monitor_install.yaml --syntax-check
ansible-playbook -i inventories/site/hosts.yaml playbooks/validate_cluster.yaml --syntax-check
~~~

## 3. Preflight validation

~~~bash
ansible-playbook -i inventories/site/hosts.yaml \
  playbooks/preflight.yaml \
  -e @~/site-vars.yaml
~~~

Correct every failure before proceeding. The playbook refuses to run with the public example values, a non-4.20.z installer, an existing installation directory, non-9000 MTU, missing private input files, or an incorrect three-master topology.

## 4. Generate manifests and Agent ISO

~~~bash
ansible-playbook -i inventories/site/hosts.yaml \
  playbooks/prepare_install.yaml \
  -e @~/site-vars.yaml
~~~

The output displays the ISO location and SHA-256 checksum. Preserve the installation directory because it contains installer state and the later kubeconfig.

## 5. Manual BMC virtual-media step

Mount the generated ISO to all three bare-metal hosts, configure virtual-media boot, and power on or reboot the hosts.

On the rendezvous host, allow the Agent console connectivity check to complete. Do not proceed if the release-image pull check fails.

When the installation has started, run:

~~~bash
ansible-playbook -i inventories/site/hosts.yaml \
  playbooks/pause_for_bmc_boot.yaml \
  -e @~/site-vars.yaml
~~~

Press Enter only after the ISO is mounted, nodes are booted, and installation has started.

## 6. Monitor installation

~~~bash
ansible-playbook -i inventories/site/hosts.yaml \
  playbooks/monitor_install.yaml \
  -e @~/site-vars.yaml
~~~

The playbook waits for bootstrap completion, waits for installation completion, and confirms that the kubeconfig exists.

## 7. Validate the cluster

~~~bash
ansible-playbook -i inventories/site/hosts.yaml \
  playbooks/validate_cluster.yaml \
  -e @~/site-vars.yaml
~~~

Expected results:

- OpenShift version is 4.20.z.
- Exactly three nodes are Ready.
- All Cluster Operators are Available and not Progressing or Degraded.
- API and wildcard ingress DNS resolve from the bastion.
- Kubeconfig is available with mode `0600`.

## One-command workflow

After site variables are complete, you can run the full workflow:

~~~bash
ansible-playbook -i inventories/site/hosts.yaml \
  playbooks/full_deployment.yaml \
  -e @~/site-vars.yaml
~~~

The run pauses at the BMC virtual-media gate.

## Repeated test deployment using the same VIPs and DNS

The same API VIP, ingress VIP, DNS records, and node IPs can be reused for a test cluster only after the previous test cluster is fully offline and the three intended root disks have been wiped.

Use a new `deployment_id` for every test iteration. This creates a new installation directory while retaining the same network identity:

~~~yaml
environment_tier: test
deployment_id: test-run-002
reuse_network_identity: true
test_hardware_reset_confirmed: true
~~~

Do not set `allow_existing_install_directory=true` to redeploy. It bypasses only the directory guard; it does not reset a cluster or wipe disks.

Every deployment also requires explicit command-line intent:

~~~bash
ansible-playbook -i inventories/site/hosts.yaml \
  playbooks/full_deployment.yaml \
  -e @~/site-vars.yaml \
  -e deployment_intent=install \
  -e confirm_cluster_name=<cluster-name> \
  -e confirm_environment_tier=test
~~~

Before ISO boot, Ansible requires the operator to type the deployment authorization phrase shown on screen. If an OpenShift API already responds on the target API VIP, preflight stops the deployment.

For QA and production, keep `reuse_network_identity: false` and provide a non-empty `change_reference` in the private site variables file.

## Kubeconfig usage

~~~bash
export KUBECONFIG=<cluster-install-dir>/auth/kubeconfig
oc get nodes
oc get clusteroperators
~~~

Never commit or share the kubeconfig.