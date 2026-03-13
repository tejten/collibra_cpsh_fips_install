# HTTPS Enablement Runbook

Run this after completing [`CPSH_INSTALL.md`](./CPSH_INSTALL.md).

Replace all placeholder values before running commands.

## 1. Start The Admin Console And Agent

```bash
/opt/collibra/console/bin/console start
/opt/collibra/agent/bin/agent start
```

## 2. Common Variables

Use your actual CPSH hostname and strong passwords. If you switch shells with `sudo -iu collibra`, re-run these exports in that shell before running `keytool`.

```bash
export CPSH_HOST="<cpsh-host-fqdn>"
export CONSOLE_KEYSTORE_PASS="<strong-console-keystore-password>"
export DGC_KEYSTORE_PASS="<strong-dgc-keystore-password>"
export BC_LIBS="/opt/collibra/security/fips/libs/bc-fips.jar:/opt/collibra/security/fips/libs/bcutil-fips.jar:/opt/collibra/security/fips/libs/bcpkix-fips.jar"
```

## 3. Enable HTTPS For The Admin Console

Prepare temp directories:

```bash
sudo -u collibra mkdir -p /opt/collibra_data/console/tmp
sudo chmod 700 /opt/collibra_data/console/tmp
sudo -u collibra mkdir -p /opt/collibra_data/console/tmp/bcfips-native
sudo chmod 700 /opt/collibra_data/console/tmp/bcfips-native
```

Switch to the `collibra` user and create the keystore:

```bash
sudo -iu collibra
export CPSH_HOST="<cpsh-host-fqdn>"
export CONSOLE_KEYSTORE_PASS="<strong-console-keystore-password>"
export DGC_KEYSTORE_PASS="<strong-dgc-keystore-password>"
export BC_LIBS="/opt/collibra/security/fips/libs/bc-fips.jar:/opt/collibra/security/fips/libs/bcutil-fips.jar:/opt/collibra/security/fips/libs/bcpkix-fips.jar"

rm -f /opt/collibra_data/console/security/console-https.bcfks
mkdir -p /opt/collibra_data/console/security

/opt/collibra/jre/bin/keytool -genkeypair \
  -alias console-https \
  -keyalg RSA -keysize 3072 \
  -sigalg SHA384withRSA \
  -dname "CN=${CPSH_HOST}" \
  -ext "SAN=dns:${CPSH_HOST}" \
  -validity 825 \
  -keystore /opt/collibra_data/console/security/console-https.bcfks \
  -storetype BCFKS \
  -storepass "${CONSOLE_KEYSTORE_PASS}" \
  -keypass "${CONSOLE_KEYSTORE_PASS}" \
  -providername BCFIPS \
  -providerpath "${BC_LIBS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -J-Djava.io.tmpdir=/opt/collibra_data/console/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/console/tmp/bcfips-native \
  -J-Djava.security.properties=/opt/collibra/security/fips/configs/fips.enabled.java.security \
  -v
```

Validate that the keystore contains a `PrivateKeyEntry`:

```bash
/opt/collibra/jre/bin/keytool -list -v \
  -keystore /opt/collibra_data/console/security/console-https.bcfks \
  -storetype BCFKS \
  -storepass "${CONSOLE_KEYSTORE_PASS}" \
  -providername BCFIPS \
  -providerpath "${BC_LIBS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -J-Djava.io.tmpdir=/opt/collibra_data/console/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/console/tmp/bcfips-native \
  -J-Djava.security.properties=/opt/collibra/security/fips/configs/fips.enabled.java.security
```

Edit `/opt/collibra_data/console/config/server.json` and add or update:

```json
"httpsConnector": {
  "port": 5404,
  "keyAlias": "console-https",
  "keyPass": "<console-keystore-password>",
  "keystorePass": "<console-keystore-password>",
  "keystoreFile": "/opt/collibra_data/console/security/console-https.bcfks"
}
```

## 4. Enable HTTPS For DGC

Still as the `collibra` user, prepare directories:

```bash
mkdir -p /opt/collibra_data/dgc/security
mkdir -p /opt/collibra_data/dgc/tmp /opt/collibra_data/dgc/tmp/bcfips-native
chmod 700 /opt/collibra_data/dgc/tmp /opt/collibra_data/dgc/tmp/bcfips-native
```

Create the DGC keystore:

```bash
rm -f /opt/collibra_data/dgc/security/dgc-https.bcfks

/opt/collibra/jre/bin/keytool -genkeypair \
  -alias dgc-https \
  -keyalg RSA -keysize 3072 \
  -sigalg SHA384withRSA \
  -dname "CN=${CPSH_HOST}" \
  -ext "SAN=dns:${CPSH_HOST}" \
  -validity 825 \
  -keystore /opt/collibra_data/dgc/security/dgc-https.bcfks \
  -storetype BCFKS \
  -storepass "${DGC_KEYSTORE_PASS}" \
  -keypass "${DGC_KEYSTORE_PASS}" \
  -providername BCFIPS \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providerpath "${BC_LIBS}" \
  -J-Djava.io.tmpdir=/opt/collibra_data/dgc/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/dgc/tmp/bcfips-native \
  -J-Djava.security.properties=/opt/collibra/security/fips/configs/fips.enabled.java.security \
  -v
```

Verify the DGC keystore:

```bash
/opt/collibra/jre/bin/keytool -list -v \
  -keystore /opt/collibra_data/dgc/security/dgc-https.bcfks \
  -storetype BCFKS \
  -storepass "${DGC_KEYSTORE_PASS}" \
  -providername BCFIPS \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providerpath "${BC_LIBS}" \
  -J-Djava.io.tmpdir=/opt/collibra_data/dgc/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/dgc/tmp/bcfips-native \
  -J-Djava.security.properties=/opt/collibra/security/fips/configs/fips.enabled.java.security \
  | egrep -i "Keystore contains|Alias name|Entry type|Owner|Issuer"
```

Exit back to a privileged shell, then back up and edit `/opt/collibra_data/dgc/config/server.json`:

```bash
exit
sudo cp -a /opt/collibra_data/dgc/config/server.json /opt/collibra_data/dgc/config/server.json.bak.$(date +%F_%H%M%S)
sudo vi /opt/collibra_data/dgc/config/server.json
```

Set or update:

```json
"httpsConnector": {
  "port": 443,
  "keyAlias": "dgc-https",
  "keyPass": "<dgc-keystore-password>",
  "keystorePass": "<dgc-keystore-password>",
  "keystoreFile": "/opt/collibra_data/dgc/security/dgc-https.bcfks"
}
```

## 5. Update The Base URL In Collibra Console

In the DGC service settings inside Collibra Console:
- Open `General settings`
- Update the Base URL to use `https`
- Use the new HTTPS port you configured

Example:

```text
https://<cpsh-host-fqdn>:443
```

Then restart the environment and validate login with the admin credentials created during installation.

## 6. Validation Checklist
- Admin console listens on `https://<cpsh-host-fqdn>:5404`
- DGC listens on `https://<cpsh-host-fqdn>:443`
- Both BCFKS keystores exist under `/opt/collibra_data`
- Base URL in Collibra Console matches the DGC HTTPS endpoint
