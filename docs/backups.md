# Backup Strategy

## Tool: Kopia
We use Kopia to create encrypted, deduplicated snapshots of our core system and Docker data.

### Source Paths
- `/source/docker_stacks`: Contains active `docker-compose.yml` files and container-specific configuration data.
- `/source/opt_stacks`: Contains custom service configurations and deployment data.
- `/source/ubuntu_etc`: Contains critical system-level configuration files (the "brain" of the OS).

### Destination/Target
- The Kopia repository is stored on the Synology NAS (10.0.0.100).

### Restore Procedure
1. Install Kopia on the new system.
2. Connect to the repository on the Synology NAS.
3. Use `kopia restore <snapshot-id> <target-path>` to recover data.