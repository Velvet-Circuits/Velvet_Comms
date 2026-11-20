# Install the server on a new machine

Install the Velvet Comms server on a new Ubuntu or Debian computer, amd64.
The archive contains the program, the pages, and `install.sh`. The computer
does not need Go, and the download does not need a GitHub account.

Replace every placeholder before you run a command. The names
`intercom.example.com` and `203.0.113.10` are examples. The installer
refuses those two strings, so a copied example cannot obtain a certificate.

| Placeholder | Meaning |
|-------------|---------|
| `intercom.example.com` | Public DNS name of this server |
| `203.0.113.10` | Public IPv4 of this server, not a private address such as `10.x` or `192.168.x` |
| `admin@example.com` | Address for certificate expiry notices |

## 1. DNS

Create an A record before you install:

- name: `intercom.example.com`
- value: `203.0.113.10`

Check it from the server, or from any other computer:

```bash
dig +short intercom.example.com A @1.1.1.1
```

The answer must be `203.0.113.10`. An AAAA record is not required.
Browsers and the studio programs will use `https://intercom.example.com`.

## 2. Firewall

Open these ports in the cloud provider panel. When `ufw` is already
enabled on the server, the installer opens the same ports there.

| Port | Use |
|------|-----|
| TCP 80 | Let's Encrypt only. The server does not listen here |
| TCP 443 | Operator panel, administration, and signaling |
| UDP 50000 | Audio |

Port 80 must be free and reachable from the Internet while the
certificate is requested. Nothing else may listen on it. Leave other
ports closed.

## 3. Download

On the server:

```bash
cd ~
curl -fL -O https://github.com/Velvet-Circuits/Velvet_Comms/releases/latest/download/updates.json
curl -fL -O https://github.com/Velvet-Circuits/Velvet_Comms/releases/latest/download/intercom-linux-amd64.tar.gz
```

`updates.json` is the signed catalog. The `server` entry names the
archive and its SHA-256 checksum. Compare that checksum with the file
you downloaded:

```bash
python3 - << 'PY'
import hashlib, json
catalog = json.load(open("updates.json"))
server = catalog["server"]
name = server["amd64"]["file"]
digest = hashlib.sha256(open(name, "rb").read()).hexdigest()
print(server["version"], name)
print("catalog ", server["amd64"]["sha256"])
print("download", digest)
print("OK" if digest == server["amd64"]["sha256"] else "MISMATCH")
PY
```

Stop if the last line is `MISMATCH`. The program checks the same
catalog signature later, when the administration page installs an update.

Unpack the archive:

```bash
tar -xzf intercom-linux-amd64.tar.gz
cd intercom-linux-amd64
```

Read [LICENSE](LICENSE) and [THIRD-PARTY.txt](THIRD-PARTY.txt) before
you install. The same files are in this folder after you unpack the
archive. Installing the program accepts those terms. MediaNews S.r.l.
grants use for the licensee's own activity, including commercial use.
Intercom 4-wire and Intercom point-to-point also ship
[libndi_licenses.txt](libndi_licenses.txt). The server archive does not
include the NDI library.

## 4. Install

Run the installer with the real DNS name, the real public IPv4, and an
email address you read:

```bash
sudo ./install.sh intercom.example.com 203.0.113.10 admin@example.com
```

The three arguments are how the installer tells a public server from a
local one. It uses them as follows:

- the DNS name is the name on the certificate and the address operators open in the browser
- the public IPv4 is the address advertised for audio
- the email receives Let's Encrypt notices when the certificate is about to expire

On a new machine the installer then:

- installs `libopus0`, `certbot`, and `openssl`
- asks Let's Encrypt for a certificate for the DNS name, using port 80
- copies `cert.pem`, `chain.pem`, `fullchain.pem`, and `privkey.pem` into `/etc/intercom/`
- creates the system user `intercom` and runs the service as that user
- writes `/etc/intercom/intercom.env` and sets the first administrator password to `velvet`
- installs the program in `/opt/intercom` and starts `intercom.service`

The first password is printed once at the end and is also stored in
`/etc/intercom/intercom.env`. The service reads that file. It can read
the certificate key and can write only `/opt/intercom/data` and
`/var/lib/intercom`.

`HTTP_PORT=0` leaves port 80 free for certificate renewal.
`ALLOWED_ORIGINS` is `https://` plus the DNS name, with no trailing
slash and no port.

## 5. Confirm

```bash
systemctl is-active intercom
sudo journalctl -u intercom -n 20 --no-pager
curl -fsS https://intercom.example.com/healthz
sudo openssl x509 -in /etc/intercom/cert.pem -noout -subject -issuer -ext subjectAltName -dates
```

The log shows the installed version. `healthz` returns `{"status":"ok","version":"..."}`.
The certificate must list `DNS:intercom.example.com`.

Ubuntu renews the certificate with `certbot.timer`. After a renewal,
`/etc/letsencrypt/renewal-hooks/deploy/intercom` copies the new files
into `/etc/intercom/`. The server reads them by itself. Keep TCP 80
open and unused.

```bash
systemctl status certbot.timer --no-pager
sudo certbot renew --dry-run
```

## 6. First sign-in

Open `https://intercom.example.com/admin.html` and sign in with `velvet`.

1. Choose a new password of at least 8 characters. `velvet` is not accepted again.
2. The page recommends an authenticator. You can turn it on later, under Security.
3. The server runs for 5 days without a license. The page shows the days left and the machine id. Activate pastes a license now. Continue opens administration.

After those 5 days, operators, guests, and endpoint audio stay closed
until a license for this machine is saved. Administration stays open.
A license is signed for one machine, one customer, and an expiry date.
That day is included. Audio continues for 7 days after it, then closes
until a new license is saved.

The operator panel is `https://intercom.example.com/`. It stays locked
until the first user exists. Create that user in administration, then
sign operators in at the panel address.

Point each studio program at `https://intercom.example.com`. A connection
is accepted only from that address. Plain HTTP is not served.

A guest link looks like `https://intercom.example.com/join.html#<code>`.
Create it from the administration page opened at the public address,
because the link uses the address of that page. The guest needs UDP
50000 for audio, the same as the operators. Anyone with the link can
join until you switch it off.

## 7. A certificate you already have

To skip Let's Encrypt, put these files in place before the first
`sudo ./install.sh` with the public name:

- `/etc/intercom/fullchain.pem`, the domain certificate followed by the intermediates
- `/etc/intercom/privkey.pem`, the matching private key

The installer sees `fullchain.pem` and does not request a certificate.
The email argument is still required. It is not sent anywhere when the
certificate is already there.

## What is stored

| Path | Contents |
|------|----------|
| `/opt/intercom/bin/intercomd` | The server |
| `/opt/intercom/web` | The pages |
| `/opt/intercom/data/intercom.json` | Users, channels, and the license |
| `/etc/intercom/intercom.env` | Ports, public address, and the first password until you change it |
| `/etc/intercom/secret.key` | Key that lets administration show account passwords. Back it up with `intercom.json` |
| `/etc/intercom/fullchain.pem` | Certificate the server presents |
| `/etc/intercom/privkey.pem` | Private key, mode 640, readable by the `intercom` group |

Copy `intercom.json` and `secret.key` together. Without the key,
sign-in still works and administration shows "Not available" for a
stored password.

Later releases are installed from the administration page. The page
reads `https://github.com/Velvet-Circuits/Velvet_Comms/releases` and
checks the signed catalog. No GitHub token is stored.
