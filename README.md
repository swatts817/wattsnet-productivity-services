# Wattsnet Productivity Services

Print servers and productivity tools for the Wattsnet homelab, including CUPS network printing and thermal printer support.

## 🖨️ Services Overview

### CUPS Print Server
- **Purpose**: Network print server for sharing printers across the network
- **Features**: Web interface, driver management, print queue management
- **URL**: https://cups.wattslabs.dev
- **Port**: 631

### Thermal Print Server
- **Purpose**: Receipt and label printing for automation workflows
- **Features**: REST API, template printing, n8n integration
- **URL**: https://thermal.wattslabs.dev
- **Ports**: 9100 (raw), 3010 (API)

## 🌟 Features

### CUPS Server
- **Network Printing**: Share any printer on the network
- **AirPrint Support**: iOS/macOS native printing
- **Driver Management**: Automatic and manual driver installation
- **Queue Management**: Monitor and manage print jobs
- **User Access Control**: Per-printer permissions

### Thermal Printer
- **Receipt Printing**: POS-style receipts
- **Label Printing**: Shipping and inventory labels
- **Template System**: Customizable print templates
- **API Access**: REST API for programmatic printing
- **n8n Integration**: Trigger prints from workflows

## 📋 Prerequisites

- Docker and Docker Compose v2+
- External Docker network `homelab`:
  ```bash
  docker network create homelab
  ```
- Printer hardware (USB or network)
- For thermal printer: ESC/POS compatible device

## 🚀 Quick Start

1. **Clone the repository:**
   ```bash
   git clone https://github.com/yourusername/wattsnet-productivity-services.git
   cd wattsnet-productivity-services
   ```

2. **Configure environment:**
   ```bash
   cp .env.example .env
   nano .env
   # Set passwords and printer IPs
   ```

3. **Create directories:**
   ```bash
   mkdir -p cups/{config,log,spool}
   mkdir -p thermal-printer/{config,templates,logs,queue}
   ```

4. **Start services:**
   ```bash
   docker-compose up -d
   ```

5. **Access CUPS interface:**
   - Navigate to https://cups.wattslabs.dev or http://localhost:631
   - Login with configured credentials
   - Add printers through the web interface

## 🔧 Configuration

### CUPS Printer Setup

1. **Access CUPS Admin:**
   ```
   https://cups.wattslabs.dev/admin
   Username: admin
   Password: [from .env]
   ```

2. **Add Network Printer:**
   - Administration → Add Printer
   - Select network printer or enter IP
   - Choose driver (or upload PPD)
   - Set default options

3. **Add USB Printer:**
   ```yaml
   # Ensure USB device is passed through
   devices:
     - /dev/usb/lp0:/dev/usb/lp0
   ```

### Thermal Printer Setup

1. **Create Dockerfile for thermal printer:**
   ```dockerfile
   # thermal-printer/Dockerfile
   FROM node:18-alpine

   WORKDIR /app

   # Install dependencies
   COPY package*.json ./
   RUN npm ci --only=production

   # Install ESC/POS library
   RUN npm install escpos escpos-usb escpos-network

   # Copy application
   COPY . .

   EXPOSE 3010 9100

   CMD ["node", "server.js"]
   ```

2. **Configure printer connection:**
   ```javascript
   // thermal-printer/config.js
   module.exports = {
     printer: {
       type: 'network',  // or 'usb'
       ip: process.env.THERMAL_PRINTER_IP,
       port: process.env.THERMAL_PRINTER_PORT
     },
     api: {
       port: process.env.API_PORT || 3010
     }
   };
   ```

## 📱 Client Configuration

### Windows
```powershell
# Add network printer
Add-Printer -ConnectionName "\\cups.wattslabs.dev\PrinterName"
```

### macOS
1. System Preferences → Printers & Scanners
2. Click "+" → IP tab
3. Address: `cups.wattslabs.dev`
4. Protocol: IPP
5. Queue: `printers/PrinterName`

### Linux
```bash
# Add IPP printer
lpadmin -p PrinterName -E -v ipp://cups.wattslabs.dev/printers/PrinterName -m everywhere
```

### iOS/Android
- Printers should appear automatically via AirPrint/Mopria
- Or manually add IPP printer with server address

## 🔐 Security

### CUPS Authentication
```bash
# Generate password hash
docker exec -it cups passwd admin

# Configure allowed users
docker exec -it cups lpadmin -p PrinterName -u allow:user1,user2
```

### SSL Configuration
```yaml
# Add SSL certificates
volumes:
  - ./cups/ssl:/etc/cups/ssl
environment:
  - CUPS_SERVER_ENCRYPTION=required
```

### Access Control
```apache
# cups/config/cupsd.conf
<Location />
  Order allow,deny
  Allow from 192.168.0.0/24
</Location>

<Location /admin>
  Require user @SYSTEM
  Order allow,deny
  Allow from 192.168.0.0/24
</Location>
```

## 📦 API Usage

### Thermal Printer API

```bash
# Print text
curl -X POST https://thermal.wattslabs.dev/api/print \
  -H "Content-Type: application/json" \
  -d '{
    "type": "text",
    "content": "Hello, World!",
    "cut": true
  }'

# Print receipt template
curl -X POST https://thermal.wattslabs.dev/api/print/receipt \
  -H "Content-Type: application/json" \
  -d '{
    "store": "Wattsnet Store",
    "items": [
      {"name": "Item 1", "price": 9.99, "qty": 2},
      {"name": "Item 2", "price": 14.99, "qty": 1}
    ],
    "tax": 2.50,
    "total": 37.47
  }'

# Print label
curl -X POST https://thermal.wattslabs.dev/api/print/label \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Shipping Label",
    "address": "123 Main St\nCity, ST 12345",
    "barcode": "1234567890",
    "qrcode": "https://track.example.com/1234567890"
  }'
```

### CUPS API
```bash
# Get printer list
curl http://cups.wattslabs.dev:631/printers

# Get job status
curl http://cups.wattslabs.dev:631/jobs

# Cancel job
cancel -h cups.wattslabs.dev job-id
```

## 🔄 Integration

### n8n Automation
```javascript
// n8n Function Node - Print receipt
const printData = {
  type: 'receipt',
  items: items,
  total: total
};

await $http.post('https://thermal.wattslabs.dev/api/print/receipt', {
  body: printData,
  headers: {
    'Content-Type': 'application/json'
  }
});
```

### Home Assistant
```yaml
# configuration.yaml
rest_command:
  print_label:
    url: https://thermal.wattslabs.dev/api/print/label
    method: POST
    headers:
      Content-Type: application/json
    payload: '{"title": "{{ title }}", "content": "{{ content }}"}'

automation:
  - alias: Print Delivery Label
    trigger:
      platform: state
      entity_id: sensor.package_delivered
    action:
      service: rest_command.print_label
      data:
        title: "Package Received"
        content: "{{ now().strftime('%Y-%m-%d %H:%M') }}"
```

## 🛠️ Troubleshooting

### CUPS Issues

```bash
# Check CUPS status
docker exec cups lpstat -t

# View error log
docker logs cups --tail 50

# Restart CUPS
docker-compose restart cups

# Clear print queue
docker exec cups cancel -a
```

### Thermal Printer Issues

```bash
# Test printer connection
nc -zv 192.168.0.60 9100

# Check API health
curl http://localhost:3010/health

# View logs
docker logs thermal-printer --tail 100

# Test print
echo "Test" | nc 192.168.0.60 9100
```

### Common Problems

**Printer not found:**
- Check network connectivity
- Verify printer IP address
- Ensure printer is on same network/VLAN

**Permission denied:**
- Check CUPS user permissions
- Verify container has USB access
- Check file permissions on config directories

**Print jobs stuck:**
```bash
# Clear all jobs
docker exec cups cancel -a -x

# Restart printer
docker exec cups cupsenable PrinterName
```

## 📊 Monitoring

### Print Statistics
```bash
# Get printer statistics
docker exec cups lpstat -p -d

# Monitor print queue
watch -n 5 'docker exec cups lpstat -o'

# Check printer status
docker exec cups lpstat -p PrinterName -l
```

### Service Health
```bash
# Check service status
docker-compose ps

# Monitor resources
docker stats cups thermal-printer

# Check logs
docker-compose logs -f
```

## 🚀 Advanced Configuration

### High Volume Setup
```yaml
# Increase CUPS limits
environment:
  - CUPS_MAX_CLIENTS=100
  - CUPS_MAX_JOBS=500
  - CUPS_MAX_LOG_SIZE=10M
```

### Load Balancing
Deploy multiple CUPS instances:
```yaml
services:
  cups1:
    extends: cups
    container_name: cups1

  cups2:
    extends: cups
    container_name: cups2
```

### Printer Clustering
```bash
# Share printer across CUPS servers
lpadmin -p SharedPrinter -E -v ipp://cups2.local/printers/Printer
```

## 📦 Backup & Recovery

### Backup Script
```bash
#!/bin/bash
# scripts/backup-printing.sh

BACKUP_DIR="/backup/printing"
DATE=$(date +%Y%m%d-%H%M%S)

# Backup CUPS configuration
docker exec cups tar czf - /etc/cups > "$BACKUP_DIR/cups-config-$DATE.tar.gz"

# Backup thermal printer templates
tar czf "$BACKUP_DIR/thermal-templates-$DATE.tar.gz" thermal-printer/templates/

echo "Backup completed: $DATE"
```

### Restore Procedure
```bash
# Restore CUPS config
docker exec -i cups tar xzf - < /backup/cups-config-20240923.tar.gz

# Restart services
docker-compose restart
```

## 📚 Resources

- [CUPS Documentation](https://www.cups.org/documentation.html)
- [ESC/POS Commands Reference](https://reference.epson-biz.com/modules/ref_escpos/index.php)
- [IPP Protocol Specification](https://www.pwg.org/ipp/)
- [AirPrint Configuration](https://support.apple.com/airprint)
- [Wattsnet Documentation](https://github.com/yourusername/wattsnet-documentation)

## 📄 License

MIT License - See LICENSE file for details

---

**Maintained by**: Wattsnet Homelab
**Location**: VM 101 - Ubuntu Docker Host (192.168.0.100)
**Network**: 192.168.0.0/24 subnet
**Integration**: n8n workflows, Home Assistant automation
**Last Updated**: September 2024