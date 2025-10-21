# Dual Raspberry Pi 5 Server Architecture

## System Components

### Primary Node: Minecraft & Home Automation Server
- **Role**: Primary server running Minecraft server and home automation tasks
- **OS**: Raspberry OS (64-bit) / Debian
- **Key Services**:
  - Minecraft Server (Paper/Purpur recommended)
  - Cron jobs for home automation
  - Network file sharing (Samba/NFS)
  - Basic monitoring and logging
- **Connectivity**: Primary network interface for LAN services

### Secondary Node: LLM Experimentation Base
- **Role**: Local LLM inference and experimentation
- **OS**: Raspberry OS Lite (64-bit) minimal install
- **Key Services**:
  - Ollama or similar LLM runtime
  - Tiny LLM models (<1B parameters)
  - MoE (Mixture of Experts) models
  - API server for LLM access
- **Connectivity**: High-speed connection to primary node

## Network Architecture
- **Physical Connection**: Direct Gigabit Ethernet connection between nodes
- **Network Configuration**:
  - Primary node: DHCP server/primary network interface
  - Secondary node: Static IP for reliable LLM service
- **Security**: Basic firewall rules, SSH key-based authentication

## Storage
- **Primary Node**:
  - High-speed microSD card (256GB+)
  - External USB 3.0 drive for Minecraft world backups
- **Secondary Node**:
  - High-speed microSD card (128GB+)
  - Optional USB SSD for model storage

## Software Stack

### Primary Node
- **Minecraft**: Paper/Purpur with optimizations
- **Automation**: Bash scripts + cron
- **Monitoring**: Prometheus + Grafana (optional)
- **File Sharing**: Samba for Windows access

### Secondary Node
- **LLM Runtime**: Ollama
- **Models**: TinyLLaMA, Phi-2, MoE variants
- **API**: FastAPI/Flask wrapper
- **Monitoring**: Basic system monitoring

## Power Management
- **Power Supply**: 27W USB-C PD power supplies for both nodes
- **UPS**: Optional uninterruptible power supply
- **Power Monitoring**: Home Assistant integration (future)

## Security Considerations
- SSH key authentication only
- Regular OS updates
- Firewall configuration
- Separate user accounts for services

## Future Enhancements
- Docker containerization
- Kubernetes cluster (k3s)
- Home Assistant integration
- Automated backup system
