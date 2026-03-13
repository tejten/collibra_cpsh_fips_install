# CPSH 2025.10 FIPS Installation Runbooks

This repository documents a single-node `CPSH 2025.10` installation with `FIPS` enabled on `RHEL 9.7`.

## Important
- Replace all sample hostnames, passwords, installer filenames, and SSH key names before running commands.

## Architecture
- VM1: CPSH on `RHEL 9.7`
- Repository database: local `PostgreSQL 14`
- CPSH components: Repository, DGC, Search, Jobserver, Management Console

## Runbooks
1. Base install, OS prep, PostgreSQL, and CPSH installer flow: [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. Post-install HTTPS enablement for the admin console and DGC: [`HTTPS_ENABLEMENT.md`](./HTTPS_ENABLEMENT.md)

## Quick Order Of Execution
1. Complete all steps in [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. Complete all steps in [`HTTPS_ENABLEMENT.md`](./HTTPS_ENABLEMENT.md)

## License

This project is licensed under the MIT License.

## Credits

Repository structure follows the same lightweight GitHub pattern as the referenced Tej Tenmattam CPSH deployment runbook repository, adapted here for the CPSH FIPS installation flow.
