# debian13-infra

Ansible-pull baseline voor Debian 13 post-install.

## Gebruik

```bash
sudo ansible-pull \
  --url https://github.com/henrydenhengst/debian13-infra.git \
  --directory /var/lib/ansible-pull \
  --checkout main \
  --accept-host-key \
  local.yml
