# CPSH Clean Uninstall And Reinstall Runbook

Run this on the CPSH host as `root`.

This runbook is based on the attached cleanup notes and is intended for cases where you want to remove an existing CPSH install and perform a fresh reinstall.

## Important
- These steps are destructive.
- Replace the installer filename before running commands.
- The PostgreSQL cleanup section drops roles and the repository database if you choose to run it.
- Prefer the backup workflow unless you explicitly want to hard-delete the install directories.

## 1. Stop CPSH Processes And Services

```bash
systemctl stop collibra-console collibra-agent || true
pkill -f '/opt/collibra' || true

systemctl stop collibra-console collibra-agent 2>/dev/null || true
systemctl disable collibra-console collibra-agent 2>/dev/null || true
```

## 2. Remove Old Service Definitions

```bash
rm -f /etc/systemd/system/collibra-console.service \
      /etc/systemd/system/collibra-agent.service \
      /etc/systemd/system/multi-user.target.wants/collibra-console.service \
      /etc/systemd/system/multi-user.target.wants/collibra-agent.service \
      /etc/init.d/collibra-console \
      /etc/init.d/collibra-agent \
      /etc/default/collibra-console \
      /etc/default/collibra-agent \
      /etc/sysconfig/collibra-console \
      /etc/sysconfig/collibra-agent

systemctl daemon-reload
systemctl reset-failed
```

Verify the units are gone:

```bash
systemctl list-unit-files | grep -i collibra || true
find /etc -maxdepth 3 -type f | grep -Ei 'collibra|dgc' || true
```

## 3. Clean The Install Directories

Recommended backup-first option:

```bash
TS=$(date +%Y%m%d_%H%M%S)
mv /opt/collibra /opt/collibra_backup_${TS}
mv /opt/collibra_data /opt/collibra_data_backup_${TS}
mkdir -p /opt/collibra /opt/collibra_data
chown -R collibra:collibra /opt/collibra /opt/collibra_data
```

Hard-delete option:

```bash
rm -rf /opt/collibra /opt/collibra_data
mkdir -p /opt/collibra /opt/collibra_data
chown -R collibra:collibra /opt/collibra /opt/collibra_data
```

Use only one of the two paths above.

## 4. Optional PostgreSQL Cleanup

Run this only if you want to fully remove the existing CPSH repository database and roles before reinstalling:

```bash
su - postgres -c "psql -p 5432 -d postgres -c \"DROP DATABASE IF EXISTS collibra_repo;\""
su - postgres -c "psql -p 5432 -d postgres -c \"DROP ROLE IF EXISTS dgc;\""
su - postgres -c "psql -p 5432 -d postgres -c \"DROP ROLE IF EXISTS collibra;\""
```

## 5. Reinstall CPSH In FIPS Mode

```bash
cd /tmp
./dgc-linux-2025.10.391.sh -- --fips
```

If you are following the same settings as the main install guide, use [`CPSH_INSTALL.md`](./CPSH_INSTALL.md) for the expected prompt values and service layout.

## 6. Quick Post-Reinstall Validation

Check the main CPSH and PostgreSQL ports:

```bash
ss -ltnp | egrep '4403|4420|4414|5432'
```

Confirm CPSH services are registered again:

```bash
systemctl list-unit-files | grep -i collibra || true
```

## 7. Next Step

If the reinstall succeeds, continue with [`HTTPS_ENABLEMENT.md`](./HTTPS_ENABLEMENT.md) to restore HTTPS configuration for the admin console and DGC.
