# Raspberry Pi 5 Dual Server Bill of Materials (BOM)

## Core Components

### Raspberry Pi Hardware
1. **Raspberry Pi 5 8GB** - $100 each × 2 = $200
   - Primary node: Minecraft server
   - Secondary node: LLM inference

2. **Official Raspberry Pi 5 Case** - $10 each × 2 = $20
   - Includes cooling fans and heatsinks

3. **High-Speed microSD Cards** - $25 each × 2 = $50
   - SanDisk Extreme 256GB (primary node)
   - SanDisk Extreme 128GB (LLM node)

### Power Supplies
4. **Raspberry Pi 5 Official Power Supply** - $15 each × 2 = $30
   - 27W USB-C PD power delivery

5. **Uninterruptible Power Supply (UPS)** - $80
   - APC Back-UPS Pro 1500VA
   - Provides graceful shutdown protection

## Networking

### High-Speed Connection
6. **Gigabit Ethernet Cable** - $15
   - Cat 6a, 10ft cable for direct Pi-to-Pi connection

### Network Infrastructure
7. **Gigabit Ethernet Switch** - $40
   - 8-port unmanaged switch
   - For connecting both Pis to home network

8. **Ethernet Patch Cables** - $20
   - Cat 6, 5-pack for network connections

## Cooling & Thermal Management

### Active Cooling
9. **Heatsinks & Fans** (included with official cases) - $0

### Additional Cooling
10. **Thermal Pads** - $10
    - For RAM and SoC cooling optimization

## Storage Expansion

### External Storage
11. **USB 3.0 SSD** - $60
    - 500GB Samsung T7 for Minecraft world backups
    - 1TB option available for $100

### Cables & Adapters
12. **USB-C to USB-A Adapter** - $10
    - For connecting external storage

## Monitoring & Control

### Remote Access
13. **USB to Serial Adapter** - $15
    - For headless setup and debugging

### Power Monitoring
14. **Smart Plug** - $25
    - TP-Link Kasa HS110 for power monitoring

## Total Estimated Cost: $575

## Optional Upgrades

### Performance Enhancements
- **NVMe SSD Adapter**: $50 (for faster storage)
- **Active Cooling Fan**: $20 (additional cooling)

### Software & Development
- **USB Flash Drive**: $15 (for OS installation)
- **Ethernet Coupler**: $10 (for cable extension)

### Backup Solutions
- **External Hard Drive**: $80 (2TB for automated backups)