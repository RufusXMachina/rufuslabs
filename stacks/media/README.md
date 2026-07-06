# Media Stack

This stack handles the automated media download and organization ecosystem.

## Prerequisites
Because this stack uses an `external` network, you must create it before starting the containers:

```bash
docker network create media_network

Deployment

To start all services in this stack:
Bash

docker compose up -d