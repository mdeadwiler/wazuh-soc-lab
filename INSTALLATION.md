# Wazuh SOC Installation Guide

## Prerequisites

### Host System Requirements
- macOS 10.15 or later
- Minimum 8GB RAM (16GB recommended)
- 50GB free disk space
- Internet connection

### Required Software
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Multipass
brew install multipass

# Install network tools
brew install nmap hydra
```

## Step-by-Step Installation

### 1. Create Virtual Machines

#### Wazuh Manager VM
```bash
multipass launch ubuntu --name wazuh-soc --memory 4G --cpus 2 --disk 20G
multipass shell wazuh-soc
```

#### Wazuh Agent VM
```bash
multipass launch ubuntu --name wazuh-soc-agent --memory 4G --cpus 2 --disk 20G
multipass shell wazuh-soc-agent
```

### 2. Configure Wazuh Manager

#### On wazuh-soc VM:
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install curl apt-transport-https lsb-release gnupg wget -y

# Add Wazuh repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

# Update package index
sudo apt update

# Install Wazuh Manager
sudo apt install wazuh-manager -y

# Start and enable Wazuh Manager
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

#### Install Wazuh Dashboard
```bash
# Install Wazuh Dashboard
sudo apt install wazuh-dashboard -y

# Configure dashboard
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard
sudo systemctl status wazuh-dashboard
```

### 3. Configure Wazuh Agent

#### On wazuh-soc-agent VM:
```bash
# Add Wazuh repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

# Update package index
sudo apt update

# Install Wazuh Agent
sudo apt install wazuh-agent -y

# Configure agent to connect to manager
sudo sed -i 's/<server>.*<\/server>/<server>192.168.64.7<\/server>/' /var/ossec/etc/ossec.conf

# Start and enable Wazuh Agent
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

### 4. Register Agent with Manager

#### On wazuh-soc (Manager):
```bash
# Add agent
sudo /var/ossec/bin/manage_agents -a

# Follow prompts:
# - Agent name: wazuh-soc-agent
# - IP: 192.168.64.8
# - ID: 001 (or auto-generated)

# Extract agent key
sudo /var/ossec/bin/manage_agents -e 001
```

#### On wazuh-soc-agent:
```bash
# Import agent key (use key from manager)
sudo /var/ossec/bin/manage_agents -i <AGENT_KEY>

# Restart agent
sudo systemctl restart wazuh-agent
```

## Verification

### Check Manager Status
```bash
# On wazuh-soc
sudo /var/ossec/bin/wazuh-control status
sudo tail -f /var/ossec/logs/ossec.log
```

### Check Agent Status
```bash
# On wazuh-soc-agent
sudo /var/ossec/bin/wazuh-control status
sudo tail -f /var/ossec/logs/ossec.log
```

### Verify Agent Connection
```bash
# On wazuh-soc
sudo /var/ossec/bin/agent_control -l
```

## Access Wazuh Dashboard

1. Open browser and navigate to: `https://192.168.64.7:5601`
2. Default credentials:
   - Username: admin
   - Password: admin (change on first login)

## Troubleshooting

### Common Issues

#### Agent Not Connecting
```bash
# Check agent configuration
sudo cat /var/ossec/etc/ossec.conf | grep server

# Check network connectivity
ping 192.168.64.7

# Check agent logs
sudo tail -f /var/ossec/logs/ossec.log
```

#### Manager Not Starting
```bash
# Check manager status
sudo systemctl status wazuh-manager

# Check manager logs
sudo tail -f /var/ossec/logs/ossec.log

# Restart manager
sudo systemctl restart wazuh-manager
```

#### Dashboard Not Accessible
```bash
# Check dashboard status
sudo systemctl status wazuh-dashboard

# Check dashboard logs
sudo journalctl -u wazuh-dashboard -f

# Restart dashboard
sudo systemctl restart wazuh-dashboard
```

## Next Steps

After successful installation:
1. Configure custom rules
2. Set up alert notifications
3. Perform security testing
4. Analyze generated logs
5. Document findings

## Security Notes

- Change default passwords immediately
- Configure firewall rules appropriately
- Enable SSL/TLS encryption
- Implement proper access controls
- Regular system updates
