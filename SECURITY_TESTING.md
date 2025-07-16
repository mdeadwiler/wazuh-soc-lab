# Ethical Security Testing Guide

## ⚠️ Important Disclaimer

This guide is for educational purposes only. All testing must be performed on systems you own or have explicit written permission to test. Unauthorized security testing is illegal and unethical.

## Legal and Ethical Requirements

### Before You Begin
- ✅ Ensure you own all systems being tested
- ✅ Test only in isolated environments
- ✅ Document all activities
- ✅ Follow responsible disclosure practices
- ✅ Comply with local laws and regulations

### Prohibited Activities
- ❌ Testing systems you don't own
- ❌ Unauthorized access attempts
- ❌ Disrupting production systems
- ❌ Data theft or exfiltration
- ❌ Malicious intent

## Testing Environment Setup

### Network Isolation
Ensure your testing environment is properly isolated:

```bash
# Verify VM network configuration
multipass exec wazuh-soc -- ip route
multipass exec wazuh-soc-agent -- ip route

# Check network isolation
ping -c 3 8.8.8.8  # Should work from host
multipass exec wazuh-soc -- ping -c 3 8.8.8.8  # Should work from VM
```

### Baseline Configuration
Before testing, document the baseline:

```bash
# Check services running
multipass exec wazuh-soc-agent -- sudo systemctl list-units --type=service --state=running

# Check open ports
multipass exec wazuh-soc-agent -- sudo netstat -tuln

# Check user accounts
multipass exec wazuh-soc-agent -- cat /etc/passwd
```

## SSH Brute Force Testing

### Preparation
```bash
# Create test wordlists (keep them small for controlled testing)
cat << EOF > userlist.txt
admin
root
ubuntu
test
user
EOF

cat << EOF > passlist.txt
password
123456
admin
root
ubuntu
EOF
```

### Network Reconnaissance
```bash
# Basic network scan
nmap -sn 192.168.64.0/24

# Service detection on agent
nmap -sV -p 22 192.168.64.8

# OS detection
nmap -O 192.168.64.8
```

### Controlled Brute Force Attack
```bash
# Install hydra if not already installed
brew install hydra

# Perform limited brute force (adjust -t for threads, -f to stop on first success)
hydra -L userlist.txt -P passlist.txt -t 4 -f 192.168.64.8 ssh

# Alternative: Manual attempts for more control
ssh admin@192.168.64.8  # Will fail
ssh root@192.168.64.8   # Will fail
ssh test@192.168.64.8   # Will fail
```

### Monitoring During Testing
```bash
# Monitor Wazuh alerts in real-time
multipass exec wazuh-soc -- sudo tail -f /var/ossec/logs/alerts/alerts.log

# Monitor system authentication logs
multipass exec wazuh-soc-agent -- sudo tail -f /var/log/auth.log

# Check agent status
multipass exec wazuh-soc-agent -- sudo /var/ossec/bin/wazuh-control status
```

## Expected Wazuh Detections

### SSH Brute Force Rules
- **Rule 5712**: SSH brute force attack
- **Rule 5710**: Multiple authentication failures
- **Rule 5720**: Multiple login failures from same source

### Sample Alert Structure
```json
{
  "timestamp": "2025-01-XX 21:XX:XX",
  "rule": {
    "level": 10,
    "id": 5712,
    "description": "SSH brute force attack detected"
  },
  "agent": {
    "id": "001",
    "name": "wazuh-soc-agent",
    "ip": "192.168.64.8"
  },
  "data": {
    "srcip": "192.168.64.1",
    "dstuser": "root",
    "dstport": "22"
  }
}
```

## Log Analysis

### Key Log Locations
```bash
# Wazuh Manager logs
multipass exec wazuh-soc -- sudo cat /var/ossec/logs/alerts/alerts.log

# Agent logs
multipass exec wazuh-soc-agent -- sudo cat /var/ossec/logs/ossec.log

# System authentication logs
multipass exec wazuh-soc-agent -- sudo cat /var/log/auth.log
```

### Analysis Commands
```bash
# Count failed SSH attempts
multipass exec wazuh-soc-agent -- sudo grep "Failed password" /var/log/auth.log | wc -l

# Show unique IPs attempting SSH
multipass exec wazuh-soc-agent -- sudo grep "Failed password" /var/log/auth.log | grep -oE "from [0-9.]+" | sort | uniq -c

# Show Wazuh rule triggers
multipass exec wazuh-soc -- sudo grep "Rule: 5712" /var/ossec/logs/alerts/alerts.log
```

## Post-Testing Cleanup

### Reset Environment
```bash
# Clear log files (optional)
multipass exec wazuh-soc-agent -- sudo truncate -s 0 /var/log/auth.log
multipass exec wazuh-soc -- sudo truncate -s 0 /var/ossec/logs/alerts/alerts.log

# Remove test files
rm userlist.txt passlist.txt

# Document findings
echo "Testing completed on $(date)" >> testing_log.txt
```

### Restart Services
```bash
# Restart Wazuh services
multipass exec wazuh-soc -- sudo systemctl restart wazuh-manager
multipass exec wazuh-soc-agent -- sudo systemctl restart wazuh-agent
```

## Documentation Requirements

### Test Report Template
```markdown
# Security Test Report

## Test Information
- **Date**: 
- **Tester**: 
- **Environment**: Wazuh SOC Lab
- **Target**: 192.168.64.8 (wazuh-soc-agent)

## Test Objectives
- Validate SSH brute force detection
- Test Wazuh alerting capabilities
- Verify log correlation

## Test Results
- **Attacks Detected**: X/Y
- **Detection Time**: Average X seconds
- **False Positives**: X
- **Alert Accuracy**: X%

## Recommendations
- [ ] Implement rate limiting
- [ ] Configure fail2ban
- [ ] Enhance monitoring rules
- [ ] Improve alert notifications
```

## Best Practices

### Testing Guidelines
1. **Start Small**: Begin with minimal test cases
2. **Document Everything**: Keep detailed logs
3. **Monitor Continuously**: Watch for unintended effects
4. **Test Incrementally**: Gradually increase complexity
5. **Clean Up**: Remove test artifacts

### Security Considerations
- Use strong passwords for legitimate accounts
- Implement proper firewall rules
- Regular security updates
- Monitor for unusual activity
- Backup configurations before testing

## Troubleshooting

### Common Issues
- **No alerts generated**: Check agent connectivity
- **Too many false positives**: Adjust rule thresholds
- **Performance issues**: Reduce test intensity
- **Network connectivity**: Verify VM network settings

### Debug Commands
```bash
# Check agent connectivity
multipass exec wazuh-soc -- sudo /var/ossec/bin/agent_control -l

# Verify rule loading
multipass exec wazuh-soc -- sudo /var/ossec/bin/wazuh-logtest

# Test configuration
multipass exec wazuh-soc -- sudo /var/ossec/bin/wazuh-control configtest
```

## Resources

- [Wazuh Ruleset Documentation](https://documentation.wazuh.com/current/user-manual/ruleset/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [Ethical Hacking Guidelines](https://www.eccouncil.org/ethical-hacking/)

---

**Remember**: With great power comes great responsibility. Use these skills to protect, not to harm.
