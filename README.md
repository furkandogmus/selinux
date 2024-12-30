# SELinux Learning Guide with Hands-on Projects

## Introduction
SELinux (Security-Enhanced Linux) is a Linux kernel security module that provides a mechanism for supporting access control security policies. Originally developed by the NSA, SELinux implements mandatory access controls (MAC) that constrain user and application access to system resources beyond the traditional Unix permissions.

### Why SELinux Matters
- Provides an additional layer of system security
- Prevents unauthorized access even if traditional security measures fail
- Limits potential damage from compromised systems
- Enforces separation of information based on confidentiality and integrity requirements

## SELinux Taxonomy
```
SELinux
├── Access Control Model
│   ├── Mandatory Access Control (MAC)
│   │   ├── Type Enforcement (TE)
│   │   ├── Role-Based Access Control (RBAC)
│   │   └── Multi-Level Security (MLS)
│   └── Discretionary Access Control (DAC)
│
├── Security Context
│   ├── User Component
│   │   ├── system_u
│   │   ├── user_u
│   │   └── root
│   ├── Role Component
│   │   ├── object_r
│   │   ├── system_r
│   │   └── staff_r
│   ├── Type Component
│   │   ├── domain types (_t)
│   │   └── resource types (_t)
│   └── Level Component
│       ├── Sensitivity (s0-s15)
│       └── Categories (c0-c1023)
│
├── Policy Types
│   ├── Targeted (Default)
│   │   ├── Confined domains
│   │   └── Unconfined domains
│   ├── Minimum
│   │   └── Limited process set
│   └── MLS
│       └── Full MAC implementation
│
├── Modes
│   ├── Enforcing
│   ├── Permissive
│   └── Disabled
│
└── Policy Rules
    ├── Allow rules
    ├── Dontaudit rules
    ├── Type transitions
    ├── Role transitions
    └── Constraints
```

### Core Concepts Explained
1. **Access Control Model**
   - MAC: Primary security mechanism
   - TE: Domain and type separation
   - RBAC: User role restrictions
   - MLS: Hierarchical security levels

2. **Security Context**
   - Identity management through user component
   - Access control through role component
   - Resource isolation through type component
   - Classification through level component

3. **Policy Framework**
   - Policy types determine scope
   - Rules define allowed operations
   - Transitions manage context changes
   - Constraints limit operations

4. **Operational States**
   - Modes control policy enforcement
   - Logging captures security events
   - Transitions handle context changes
   - Labels maintain security attributes

## Prerequisites
- Linux system (RHEL, CentOS, or Fedora recommended)
- Root access
- Basic Linux command line knowledge
- Understanding of basic security concepts
- Familiarity with Linux file permissions

### System Preparation
```bash
# Install SELinux management tools
dnf install selinux-policy-devel setools-console policycoreutils-python-utils

# Verify SELinux is installed
rpm -qa | grep selinux
```

## Part 1: SELinux Basics
**Theory:**
Understanding SELinux fundamentals is crucial for effective security management.

### Mandatory Access Control (MAC)
- Unlike Discretionary Access Control (DAC), MAC enforces system-wide policies
- Policies are enforced regardless of user permissions
- Every process, file, and system resource has a security context
- Access decisions are based on security contexts and policy rules

### SELinux Modes Explained
1. **Enforcing Mode**
   - Full SELinux protection
   - Access denied by policy is blocked and logged
   - Default and recommended for production

2. **Permissive Mode**
   - Policy violations are logged but not enforced
   - Useful for testing and troubleshooting
   - Helps identify required policy changes

3. **Disabled Mode**
   - SELinux is completely disabled
   - No protection or logging
   - Not recommended for production

**Project: SELinux Status Explorer**
```bash
# Check SELinux status and detailed information
sestatus

# View current enforcement mode
getenforce

# Switch between modes (requires root)
setenforce 1  # Enforcing
setenforce 0  # Permissive

# View SELinux boot parameters
cat /etc/selinux/config

# View SELinux decisions in real time
ausearch -m AVC -ts recent

# Check process contexts
ps auxZ | grep httpd
```

### Understanding the Output
- sestatus shows:
  - Current mode
  - Policy type
  - Policy version
  - Policy from config file
  - File contexts status

## Part 2: SELinux Contexts
**Theory:**
Security contexts are labels assigned to every system object, defining their security attributes.

### Context Components
1. **User**: SELinux user identity (different from Linux user)
   - system_u: System processes
   - user_u: User processes
   - root: Administrative processes

2. **Role**: SELinux role (part of Role-Based Access Control)
   - object_r: For files and processes
   - system_r: For system processes
   - staff_r: For administrative tasks

3. **Type**: Primary mechanism for enforcing policy
   - Most specific part of the context
   - Defines what resources can interact
   - Format: name_t (ends with _t)

4. **Level**: MLS/MCS security level
   - Optional in targeted policy
   - Format: s0-s15:c0.c1023

**Project: Web Server Context Management**
```bash
# Install Apache
dnf install httpd

# Create custom content directory
mkdir -p /custom/webroot
echo "<html><body>Hello SELinux</body></html>" > /custom/webroot/index.html

# View current context
ls -Z /custom/webroot

# List default Apache contexts
semanage fcontext -l | grep httpd

# Fix context for web content
semanage fcontext -a -t httpd_sys_content_t "/custom/webroot(/.*)?"
restorecon -Rv /custom/webroot

# Verify Apache can read the content
systemctl start httpd
curl http://localhost

# Monitor for denials
tail -f /var/log/audit/audit.log | grep httpd
```

### Common Context Operations
```bash
# Temporary context change
chcon -t httpd_sys_content_t file.html

# Copy with context
cp -Z source dest

# View process contexts
ps -eZ

# View user mappings
semanage login -l
```

## Part 3: SELinux Policies
**Theory:**
Policies define the rules that control access between processes and resources.

### Policy Types
1. **Targeted Policy (Default)**
   - Protects selected system services
   - Other processes run unconfined
   - Balance between security and usability

2. **Minimum Policy**
   - Protects only selected processes
   - Smaller policy size
   - Used in containers/embedded systems

3. **MLS (Multi-Level Security)**
   - Full mandatory access control
   - Used in high-security environments
   - Complex configuration required

**Project: Custom Policy Creation**
```bash
# Install policy development tools
dnf install policycoreutils-python-utils selinux-policy-devel

# Create a test service
cat > /usr/local/bin/myservice.sh << 'EOF'
#!/bin/bash
while true; do
    echo "Service running" >> /var/log/myservice.log
    sleep 60
done
EOF
chmod +x /usr/local/bin/myservice.sh

# Monitor for denials
ausearch -m AVC -ts recent

# Generate policy from denials
audit2allow -M myservice_policy -i /var/log/audit/audit.log

# Review generated policy
cat myservice_policy.te

# Install policy
semodule -i myservice_policy.pp

# Verify policy installation
semodule -l | grep myservice
```

### Policy Development Best Practices
1. Start in permissive mode
2. Test thoroughly
3. Review generated rules
4. Minimize policy scope
5. Document policy changes

## Part 4: File and Port Labels
**Theory:**
File and port labels control access to resources and network services.

### File Context Rules
- Persistent across file system operations
- Inherited from parent directories
- Can be customized per directory tree
- Applied during file system relabeling

### Port Label Management
- Controls which services can bind to ports
- Prevents service impersonation
- Enables non-standard port usage

**Project: Secure File Server**
```bash
# Install vsftpd
dnf install vsftpd

# Create secure FTP directory structure
mkdir -p /var/ftp/{pub,private,incoming}

# List default FTP contexts
semanage fcontext -l | grep ftp

# Set appropriate contexts
semanage fcontext -a -t public_content_t "/var/ftp/pub(/.*)?"
semanage fcontext -a -t public_content_rw_t "/var/ftp/incoming(/.*)?"
semanage fcontext -a -t private_content_t "/var/ftp/private(/.*)?"

# Apply contexts
restorecon -Rv /var/ftp

# Configure vsftpd
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.orig
cat >> /etc/vsftpd/vsftpd.conf << 'EOF'
anon_upload_enable=YES
anon_mkdir_write_enable=YES
allow_ftpd_full_access=YES
EOF

# Set boolean for FTP home directories
setsebool -P ftp_home_dir on

# Start service
systemctl start vsftpd
systemctl enable vsftpd

# Test access
curl ftp://localhost/pub/
```

## Part 5: Network Controls
**Theory:**
SELinux network controls protect services and enforce network boundaries.

### Network Context Types
- Port types (http_port_t, ssh_port_t)
- Network packet labels
- Network interface labels
- Network node labels

### Common Network Operations
**Project: Custom Service Port**
```bash
# View all port definitions
semanage port -l

# Add custom HTTP port
semanage port -a -t http_port_t -p tcp 8080

# Configure Apache for custom port
sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf

# Verify port label
semanage port -l | grep http_port_t

# Test custom port
systemctl restart httpd
curl http://localhost:8080

# Remove custom port (if needed)
semanage port -d -t http_port_t -p tcp 8080
```

### Network Troubleshooting
```bash
# Check network contexts
netstat -Z

# View network rules
sesearch --allow | grep netif

# Monitor network denials
ausearch -m AVC -c httpd
```

## Part 6: Troubleshooting
**Theory:**
Effective troubleshooting is crucial for maintaining SELinux-enabled systems.

### Common Issues and Solutions
1. **File Context Issues**
   - Wrong context preventing access
   - Missing context after file creation
   - Context lost after moving files

2. **Policy Violations**
   - Denied operations
   - Missing policy rules
   - Boolean settings

3. **Network Access Problems**
   - Port labeling issues
   - Network packet labeling
   - Service access controls

**Project: SELinux Detective**
```bash
# Create troubleshooting script
cat > selinux_detective.sh << 'EOF'
#!/bin/bash
echo "=== SELinux Status ==="
sestatus

echo -e "\n=== Recent Denials ==="
ausearch -m AVC -ts recent

echo -e "\n=== Boolean Settings ==="
getsebool -a | grep httpd

echo -e "\n=== File Context Issues ==="
find / -context "*:system_u:object_r:default_t:*" -type f 2>/dev/null

echo -e "\n=== Policy Modules ==="
semodule -l
EOF
chmod +x selinux_detective.sh

# Run troubleshooting
./selinux_detective.sh
```

### Advanced Troubleshooting
```bash
# Generate detailed report
sealert -a /var/log/audit/audit.log

# Analyze policy coverage
sepolgen-ifgen

# Check file context differences
findcon -d /var
```

## Part 7: Advanced Topics
**Theory:**
Advanced SELinux features provide fine-grained security controls.

### Multi-Level Security (MLS)
- Hierarchical security levels
- Non-hierarchical categories
- Information flow control
- Military-grade security

### Multi-Category Security (MCS)
- Horizontal separation
- Category-based access control
- Container isolation
- Cloud security

**Project: MLS Environment**
```bash
# Switch to MLS policy
sed -i 's/SELINUXTYPE=targeted/SELINUXTYPE=mls/' /etc/selinux/config

# Create MLS users and roles
semanage user -a -R "staff_r sysadm_r" -r "s0-s15:c0.c1023" staff_u

# Create test directories with different levels
mkdir -p /data/{public,confidential,secret,topsecret}

# Set MLS labels
chcon -l s0 /data/public
chcon -l s1 /data/confidential
chcon -l s2 /data/secret
chcon -l s3 /data/topsecret

# Create test users
useradd -Z staff_u test_user1
useradd -Z staff_u test_user2

# Set user ranges
semanage login -a -s staff_u -r s0-s1 test_user1
semanage login -a -s staff_u -r s0-s3 test_user2
```

### Custom Policy Module Development
```bash
# Create policy template
sepolicy generate --init myapp

# Edit policy
vim myapp.te

# Build and install
make -f /usr/share/selinux/devel/Makefile
semodule -i myapp.pp
```

## Real-World Implementation Guide

### 1. Container Security
**Use Case:** Securing containerized applications
```bash
# Set up container contexts
semanage fcontext -a -t container_file_t "/var/lib/containers(/.*)?"
semanage fcontext -a -t container_runtime_t "/usr/bin/podman"

# Configure container networking
semanage port -a -t container_port_t -p tcp 8080-8090

# Enable container features
setsebool -P container_manage_cgroup 1
setsebool -P container_use_devices 1

# Verify container isolation
ps -eZ | grep container
```

### 2. Web Application Stack
**Use Case:** Securing LAMP stack
```bash
# Configure web server
semanage fcontext -a -t httpd_sys_content_t "/var/www/html(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/uploads(/.*)?"
setsebool -P httpd_can_network_connect_db 1
setsebool -P httpd_can_sendmail 1

# Database security
semanage fcontext -a -t mysqld_db_t "/var/lib/mysql(/.*)?"
semanage port -a -t mysqld_port_t -p tcp 3306

# PHP configuration
setsebool -P httpd_execmem 1
setsebool -P httpd_can_network_connect 1
```

### 3. File Sharing Service
**Use Case:** Secure NFS/Samba setup
```bash
# NFS configuration
semanage fcontext -a -t nfs_t "/exports(/.*)?"
setsebool -P nfs_export_all_rw 1

# Samba shares
semanage fcontext -a -t samba_share_t "/srv/samba(/.*)?"
setsebool -P samba_export_all_rw 1
setsebool -P samba_create_home_dirs 1
```

### 4. Custom Application Deployment
**Use Case:** Deploying in-house applications
```bash
# Application directory setup
semanage fcontext -a -t custom_app_t "/opt/myapp(/.*)?"
semanage port -a -t custom_app_port_t -p tcp 9000

# Create custom policy module
cat > myapp.te << 'EOF'
module myapp 1.0;

require {
    type custom_app_t;
    type http_port_t;
    class tcp_socket name_connect;
}

allow custom_app_t http_port_t:tcp_socket name_connect;
EOF

# Build and install policy
make -f /usr/share/selinux/devel/Makefile myapp.pp
semodule -i myapp.pp
```

### 5. Monitoring and Logging
**Use Case:** Security monitoring setup
```bash
# Configure log directories
semanage fcontext -a -t auditd_log_t "/var/log/audit(/.*)?"
semanage fcontext -a -t syslog_t "/var/log/messages"

# Set up monitoring daemon
cat > monitor.sh << 'EOF'
#!/bin/bash
while true; do
    ausearch -m AVC -ts recent | \
    grep -v "denied  { getattr }" | \
    grep -v "denied  { read }" >> /var/log/selinux_alerts.log
    sleep 300
done
EOF
chmod +x monitor.sh

# Create systemd service
cat > /etc/systemd/system/selinux-monitor.service << 'EOF'
[Unit]
Description=SELinux Monitoring Service
After=auditd.service

[Service]
ExecStart=/path/to/monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### 6. Network Service Protection
**Use Case:** Securing network services
```bash
# SSH configuration
semanage port -a -t ssh_port_t -p tcp 2222
semanage login -a -s staff_u admin_user

# VPN setup
semanage port -a -t openvpn_port_t -p udp 1194
setsebool -P openvpn_can_network_connect 1

# Load balancer configuration
semanage port -a -t http_port_t -p tcp 8443
setsebool -P haproxy_connect_any 1
```

### 7. Development Environment
**Use Case:** Setting up secure development environment
```bash
# IDE and tools
semanage fcontext -a -t usr_t "/opt/development(/.*)?"
setsebool -P deny_ptrace 0
setsebool -P domain_can_mmap_files 1

# Git repository
semanage fcontext -a -t git_sys_content_t "/var/git(/.*)?"
restorecon -Rv /var/git

# Build environment
semanage fcontext -a -t usr_t "/opt/build(/.*)?"
setsebool -P selinuxuser_execmod 1
```

### Common Issues and Solutions

#### 1. Permission Denied Despite Correct Unix Permissions
```bash
# Check SELinux denials
ausearch -m AVC -ts recent

# Verify context
ls -Z /path/to/file

# Fix context if needed
restorecon -Rv /path/to/file

# Or add custom rule
semanage fcontext -a -t appropriate_type_t "/path/to/file"
```

#### 2. Service Won't Start
```bash
# Check service status with context
systemctl status service_name -l

# View recent denials for service
ausearch -m AVC -c "service_name"

# Generate policy if needed
audit2allow -M myservice < /var/log/audit/audit.log
```

#### 3. Network Connection Issues
```bash
# Check port labels
semanage port -l | grep service_name

# Allow network connection
setsebool -P service_name_connect_any 1

# Add custom port
semanage port -a -t service_port_t -p tcp PORT_NUMBER
```

## Common Commands Reference
```bash
# Context Management
ls -Z                  # View file contexts
chcon                  # Change context temporarily
restorecon             # Restore default context
semanage fcontext      # Manage file contexts
matchpathcon          # Show default context

# Policy Management
semodule -l            # List modules
semodule -i            # Install module
semodule -r            # Remove module
sepolicy generate     # Generate policy template
audit2allow          # Generate policy from denials

# Troubleshooting
ausearch               # Search audit logs
aureport              # Generate summary reports
sealert               # Analyze security events
seinfo                # Policy query tool
sesearch              # Search policy rules
```

## Best Practices
1. Always start in permissive mode when testing new configurations
2. Use audit2allow only after understanding the security implications
3. Document all policy changes and maintain change log
4. Regular audit log monitoring and analysis
5. Keep SELinux enabled and in enforcing mode in production
6. Backup policy modules before making changes
7. Test changes in development environment first
8. Use minimal required permissions
9. Regular security audits
10. Keep policies and tools updated

## Additional Resources
- [Red Hat SELinux Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/index)
- [SELinux Project Wiki](https://selinuxproject.org/page/Main_Page)
- [SELinux User Resources](https://github.com/SELinuxProject/selinux)
- [SELinux Policy Development Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/index)
- [SELinux Coloring Book](https://people.redhat.com/duffy/selinux/selinux-coloring-book_A4-Stapled.pdf)

## Contributing
Feel free to contribute to this guide by:
- Adding new projects and examples
- Improving existing documentation
- Sharing troubleshooting tips and solutions
- Suggesting best practices from real-world deployments
- Reporting issues and corrections
- Adding more advanced topics and use cases

## License
This guide is licensed under the MIT License.

## Changelog
- 2024-02-XX: Initial comprehensive guide
- Added detailed explanations for each section
- Included practical projects and examples
- Enhanced troubleshooting section
- Added advanced topics and best practices
