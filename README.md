# homelab-elasticsearch

Ansible playbooks and roles to install, configure, and manage Elasticsearch on Ubuntu 22.04.

## Roles

- `elasticsearch_install`: Installs necessary packages and Elasticsearch.
- `elasticsearch_configure`: Configures network and security settings.
- `elasticsearch_service`: Ensures Elasticsearch starts automatically.
- `elasticsearch_heapsize`: Configures the heap size.

## Playbook

The main playbook `main.yaml` includes all roles:

```yaml
---
- name: Install and configure Elasticsearch on Ubuntu 22.04
  hosts: elasticsearch
  become: yes
  roles:
    - elasticsearch_install
    - elasticsearch_configure
    - elasticsearch_service
    - elasticsearch_heapsize

  handlers:
    - name: Restart Elasticsearch
      systemd:
        name: elasticsearch
        state: restarted

Usage
Run the playbook with:

ansible-playbook -i <inventory_file> [main.yaml](http://_vscodecontentref_/1)

License
MIT License