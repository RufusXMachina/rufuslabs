graph TD
    User((User)) -- Internet --> Cloudflare[Cloudflare Tunnel]
    Cloudflare -- Secure Tunnel --> Ubuntu[Ubuntu Server]
    
    subgraph "Local Resources"
        Ubuntu -- USB/Direct --> DAS[TerraMaster DAS - Active Media]
        Ubuntu -- Network --> NAS[Synology NAS - Backup Only]
    end
    
    subgraph "Data Protection"
    Ubuntu -- Kopia --> NAS
    end