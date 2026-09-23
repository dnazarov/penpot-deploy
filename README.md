# Penpot Deploy

Ansible deployment of Penpot to Docker Swarm with an external NGINX reverse proxy/load balancer.

## Architecture

```
DNS
 |
 v
External NGINX :443
 |
 +--> swarm01:8080
 +--> swarm02:8080
 +--> swarm03:8080
 +--> swarm04:8080
        |
        v
 Docker Swarm routing mesh
        |
        v
 penpot-frontend
      /      \
     v        v
 backend   exporter
    |
 +--+---------+
 |            |
PostgreSQL   Valkey
```

The external NGINX terminates TLS and balances traffic between the published port on Swarm nodes. PostgreSQL and Penpot file assets are pinned to the node labeled `penpot_storage=true`.

## Repository layout

```
.
├── group_vars/
│   └── all.yml
├── inventory/
│   └── production.yml
├── nginx/
│   └── penpot.conf.example
├── playbooks/
│   └── deploy-penpot.yml
├── roles/
│   └── penpot/
│       ├── tasks/
│       │   └── main.yml
│       └── templates/
│           └── penpot-stack.yml.j2
├── vault/
│   └── secrets.example.yml
├── .gitignore
└── requirements.yml
```

## Requirements

Install Ansible and the Docker collection on the control host:

```bash
ansible-galaxy collection install -r requirements.yml
```

Docker must already be installed and Docker Swarm initialized. The first host in the `swarm_managers` group is used to manage node labels and deploy the stack.

The Swarm manager also needs the Python packages required by `community.docker.docker_stack`:

```bash
python3 -m pip install jsondiff pyyaml
```

## 1. Configure inventory

Edit:

```
inventory/production.yml
```

Set the real IP addresses/hostnames of the Swarm nodes.

The host in the `penpot_storage` group is used for PostgreSQL and local Penpot assets.

## 2. Configure application variables

Edit:

```
group_vars/all.yml
```

At minimum set:

- `penpot_public_uri`
- `penpot_storage_node`
- `penpot_publish_port`
- Penpot/PostgreSQL versions as required.

## 3. Create encrypted secrets

Copy the example:

```bash
cp vault/secrets.example.yml vault/secrets.yml
```

Generate a Penpot secret key:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(64))"
```

Edit `vault/secrets.yml`, then encrypt it:

```bash
ansible-vault encrypt vault/secrets.yml
```

Do not commit an unencrypted secrets file.

## 4. Deploy

```bash
ansible-playbook \
  -i inventory/production.yml \
  playbooks/deploy-penpot.yml \
  --ask-vault-pass
```

Check the deployment on a Swarm manager:

```bash
docker stack services penpot
docker service ls
```

View logs:

```bash
docker service logs -f penpot_penpot-backend
docker service logs -f penpot_penpot-frontend
```

## 5. External NGINX

Copy `nginx/penpot.conf.example` to the external NGINX host and replace the example IP addresses and domain.

The DNS A record should point to the external NGINX/VIP, not directly to the Penpot container.

Example:

```
penpot.company.ru A 10.10.10.100
```

NGINX then balances traffic to the published Swarm port on the cluster nodes.

## Storage note

The supplied stack uses local bind mounts for PostgreSQL and Penpot assets and pins the relevant services to a storage node.

For HA deployments, move Penpot object storage to an S3-compatible service such as MinIO and use externally managed/HA PostgreSQL. This removes the local-assets dependency from frontend/backend services and makes horizontal scaling easier.
