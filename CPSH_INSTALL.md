# CPSH 2025.10 FIPS Install Runbook

Run this on the CPSH VM (`RHEL 9.7`) as `root`, unless a step explicitly switches to another user.

## 1. Prerequisites
- A `RHEL 9.7` host for CPSH
- Root or sudo access
- CPSH installer bundle available, for example `dgc-linux-2025.10.391.sh`
- Network access to the host over SSH
- Replace all placeholder values such as `<cpsh-host>` and `<ssh-key.pem>`

## 2. Enable FIPS Mode

```bash
sudo fips-mode-setup --enable
sudo reboot
```

After reboot, verify the host is in FIPS mode:

```bash
sudo fips-mode-setup --check
update-crypto-policies --show
```

## 3. Set SELinux To Permissive


```bash
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo more /etc/selinux/config
sudo setenforce 0
```

## 4. Create The `collibra` User And System Limits

```bash
sudo useradd -m -s /bin/bash collibra

sudo tee /etc/security/limits.d/collibra.conf >/dev/null <<'EOF'
collibra soft nofile 65536
collibra hard nofile 65536
collibra soft nproc 4096
collibra hard nproc 4096
collibra soft fsize unlimited
collibra hard fsize unlimited
collibra soft as unlimited
collibra hard as unlimited
EOF

sudo tee /etc/sysctl.d/99-collibra.conf >/dev/null <<'EOF'
vm.max_map_count = 262144
EOF

sudo sysctl --system
```

## 5. Prepare The Repository Database

Clean packages and install the PostgreSQL repo:

```bash
yum clean all && yum update -y
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-$(rpm -E %{rhel})-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf repolist --all | grep -i pgdg
dnf config-manager --set-disabled pgdg18
dnf config-manager --set-disabled pgdg17
dnf config-manager --set-enabled pgdg14
dnf clean all
dnf -y update
yum -y install postgresql14 postgresql14-server postgresql14-contrib
```

If `dnf config-manager` is unavailable, install `dnf-plugins-core` first.

Update the runtime directory permissions in `/usr/lib/tmpfiles.d/postgresql-14.conf`:

```text
d /run/postgresql 2777 postgres postgres - -
```

```bash
stat -c '%a %U %G %n' /run/postgresql
```

Expected output should show `2777 postgres postgres /run/postgresql`.

Initialize PostgreSQL 14 and update the locale:

```bash
echo 'LC_ALL="en_US.UTF-8"' >> /etc/locale.conf
/usr/pgsql-14/bin/postgresql-14-setup initdb
systemctl enable postgresql-14
systemctl start postgresql-14
```

Edit `/var/lib/pgsql/14/data/postgresql.conf` and set:

```text
listen_addresses = '*'
max_connections = 500
shared_buffers = 256MB
```

Then restart PostgreSQL:

```bash
systemctl restart postgresql-14.service
```

## 6. Transfer The CPSH Installer

Example transfer and login flow:

```bash
scp -i <ssh-key.pem> dgc-linux-2025.10.391.sh ec2-user@<cpsh-host>:/tmp/
ssh -i <ssh-key.pem> ec2-user@<cpsh-host>
cd /tmp
chmod a+x dgc-linux-2025.10.391.sh
sudo ./dgc-linux-2025.10.391.sh -- --fips
```

## 7. Installer Answers 

Use the defaults below unless your environment requires different values.

| Prompt | Value |
| --- | --- |
| Installation directory | `/opt/collibra` |
| Data directory | `/opt/collibra_data` |
| Install DGC | `Y` |
| Install Repository | `Y` |
| Install Jobserver | `Y` |
| Install Search | `Y` |
| Install Management Console | `Y` |
| PostgreSQL path | `/usr/pgsql-14` |
| Repository port | `4403` |
| DGC port | `4400` |
| DGC shutdown port | `4430` |
| DGC min memory | `1024` |
| DGC max memory | `2048` |
| Agent port | `4401` |
| Agent address | `localhost` |
| Search HTTP port | `4421` |
| Search transport port | `4422` |
| Search memory | `1024` |
| Jobserver port | `4404` |
| Jobserver database port | `4414` |
| Management Console port | `4402` |
| Management Console database port | `4420` |

Keep a secure record of the repository admin password and repository DGC password that the installer requests.

## 8. Expected Successful End State


```text
COMPLETED - Installation finished
```

After installation, start the admin console and agent:

```bash
/opt/collibra/console/bin/console start
/opt/collibra/agent/bin/agent start
```

## 9. Next Step

Continue with [`HTTPS_ENABLEMENT.md`](./HTTPS_ENABLEMENT.md) to configure BCFKS-based HTTPS for the admin console and DGC.
