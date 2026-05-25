# Ansible Role: ansible_docker_it_tools

Deploy IT-Tools, a collection of useful IT utilities in a web interface.

## Description

This role deploys [IT-Tools](https://it-tools.tech/), a collection of handy tools for developers and IT professionals. IT-Tools provides a web-based interface for common tasks like encoding/decoding, hashing, text manipulation, and more.

## Requirements

- Ansible 2.15+
- Target OS: Ubuntu 20.04+, Debian 11+
- Docker and Docker Compose installed
- `community.docker` collection

## Role Variables

### Required Variables

```yaml
# IT-Tools web interface hostname
ittools_hostname: "tools.example.com"
```

### Optional Variables

```yaml
# Docker image version
ittools_docker_image: "corentinth/it-tools:latest"

# Port mapping
ittools_port: 80
```

## Dependencies

- `ansible_docker` - Base docker deployment pattern

## Example Playbook

```yaml
---
- name: Deploy IT-Tools
  hosts: utility_servers
  become: true
  roles:
    - ansible_common
    - ansible_docker_it_tools
```

## What This Role Does

1. Creates IT-Tools directory
2. Deploys IT-Tools via Docker Compose
3. Starts IT-Tools container

## Available Tools

IT-Tools includes utilities for:

### Encoders/Decoders
- Base64 encode/decode
- URL encode/decode
- HTML entities
- JWT decoder

### Hash & Crypto
- MD5, SHA-1, SHA-256, SHA-512
- HMAC generator
- Password generator
- UUID generator

### Text Manipulation
- Text diff
- Case converter
- String utilities
- Markdown preview

### Network Tools
- IP subnet calculator
- MAC address lookup
- Port scanner info

### Converters
- JSON to YAML
- XML formatter
- SQL formatter
- Color converter

### And More
- QR code generator
- Cron expression parser
- Docker run to compose
- Many others!

## Tags

- `it-tools` - IT-Tools tasks
- `utilities` - Utility tasks

## Access

Navigate to `https://{ittools_hostname}` to access the tool collection.

## Privacy

IT-Tools runs entirely client-side - all processing happens in your browser. No data is sent to external servers.

## License

MIT

## Author

Dan Wallace
