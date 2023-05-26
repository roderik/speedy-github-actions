![Speedy GitHub Actions](./header.jpg)

We make heavy use of Github Actions and we want to run them on our own hardware. This repo contains the Ansible playbook to setup the servers.

This repo provides:

- optimal base operating system
- Actuated runner
- Verdaccio NPM cache
- Docker registry caches for Docker Hub and Github Container Registry
- S3 server for file caching

## Actuated

Get yourself a license from [Actuated](https://docs.actuated.dev/). They provide the orchestration bits of this setup.

## Hardware

Get at least two servers on Hetzner (or somewhere else, but we use Hetzner) with at least NVME 2 disks:

- [AX 161](https://www.hetzner.com/dedicated-rootserver/ax161) AMD server
- [RX 170](https://www.hetzner.com/dedicated-rootserver/rx170) ARM server

SSH into them and take note of the Hardware data section

```
   CPU1: AMD Ryzen 9 5950X 16-Core Processor (Cores 32)
   Memory:  128754 MB
   Disk /dev/nvme0n1: 3840 GB (=> 3576 GiB)
   Disk /dev/nvme1n1: 3840 GB (=> 3576 GiB)
   Total capacity 7153 GiB with 2 Disks
```

Next, run `installimage` and pick Ubuntu 22.04.

In the config file:

- Set SWRAID to 1 and pick RAID 1
- Set the hostname to actuated-<arch>-<0 prefixed nr>
- Setup the partitions like this:

```
PART /boot ext3     1024M
PART lvm   actuated all

LV actuated swap swap swap 64G
LV actuated root /    ext4 1024G
```

Save, quit, reboot and wait for the server to come back up.

Make sure to use `parted` to remove every partition from `/dev/nvme1n1` if you are reusing a server.

## Network

Create a domainname (we use Cloudflare) and point it to the server.

## Software

In the infra folder, remove the secret.yml file from the repo and generate a new one:

```
ansible-vault create secret.yml
```

Add the following content:

```
license_key: xxx
dockerhub_pat: xxx
github_pat: xxx
```

In the inventory.yaml, add your servers in the correct arch group and make sure your ssh key is copied there, you can do so with the following command:

```
ssh-copy-id root@actuated-<arch>-<0 prefixed nr>.mydomain.net
```

Run the playbook:
```
ansible-playbook -i inventory.yaml -u root playbook.yaml --ask-vault-pass
```

> Note that the `convert logical volume to thinpool` task can take quite a while

## Github Actions

To make full use of all the caching options, you can use these actions:

### Verdaccio NPM Cache

Pipes all npm, yarn and pnpm installs through its proxy cache.

```
    - name: Setup NPM Cache
      uses: roderik/speedy-github-actions/actions/setup-npm-cache@main
```

### Docker Cache

Pipes all docker pulls through its proxy cache.

```
    - name: Setup Docker Cache
      uses: roderik/speedy-github-actions/actions/setup-docker-cache@main

    - name: Setup Docker Buildx Cache
      uses: roderik/speedy-github-actions/actions/setup-docker-buildx-cache@main
```

### File Caching

Very similar to `actions/cache`:

```
    - name: Get pnpm store directory
      id: pnpm-cache
      shell: bash
      run: |
        echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT

    - name: Cache the PNPM store
      uses: roderik/speedy-github-actions/setup-file-cache@main
      with:
        path: |
          ${{ steps.pnpm-cache.outputs.STORE_PATH }}
        key: ${{ runner.os }}-pnpm-store-${{ hashFiles('pnpm-lock.yaml') }}
        restore-keys: |
          ${{ runner.os }}-pnpm-store-
```
