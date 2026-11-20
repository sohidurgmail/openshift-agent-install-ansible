# Safety and secret-handling guidance

## Private files

Keep these files outside the repository:

- Pull secret.
- SSH private and public keys.
- Artifactory CA certificate, if it is site-specific.
- BMC credentials.
- Site-specific variables.
- Generated ISO files.
- Installer metadata and logs.
- Kubeconfig and kubeadmin password.

The playbooks read the pull secret, SSH public key, and CA from local paths. They do not create copies in the repository.

## Installation-directory safety

Use a new, empty `cluster_install_dir` for each new cluster deployment. The preflight playbook stops if the directory already exists, unless `allow_existing_install_directory=true` is explicitly supplied.

Do not enable that override unless you have verified that the directory is disposable and contains no credentials or deployment state that must be preserved.

## Pre-push review

Before pushing:

~~~bash
git status
git diff --cached
rg -n -i 'password|token|secret|kubeconfig|private key|BEGIN CERTIFICATE|10\.|172\.16|192\.168' .
~~~

Investigate every match. A public-safe repository should contain only documentation, templates, example domains, and reserved example IP ranges.
