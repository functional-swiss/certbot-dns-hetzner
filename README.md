# Let's Encrypt x Hetzner: Certbot DNS-01 challenge

> _forked from <https://github.com/dschoeffm/hetzner-dns-certbot>_

A single bash script that handles DNS-01 challenges for [certbot](https://certbot.eff.org/) using the [Hetzner Cloud DNS API](https://docs.hetzner.cloud/).

**v2** uses the new Hetzner Cloud API (`api.hetzner.cloud`). For the legacy DNS Console API (`dns.hetzner.com`), see the [`v1` tag](https://github.com/functional-swiss/certbot-dns-hetzner/tree/v1).

## Install

**Dependencies**: `curl`, `jq`, `dnsutils` (for `dig`), `certbot`

```bash
apt update && apt install -y curl jq dnsutils certbot
```

**Download the hook script**:

```bash
curl -sL "https://github.com/functional-swiss/certbot-dns-hetzner/raw/v2-console-api/certbot-hook-hetzner" \
  > /usr/local/bin/certbot-hook-hetzner && chmod +x /usr/local/bin/certbot-hook-hetzner
```

**Configure the API token**:

1. Go to [console.hetzner.com](https://console.hetzner.com) > **Security** > **API Tokens**
2. Generate a token with **Read & Write** permissions
3. Save it on the server:

```bash
echo 'your_hetzner_cloud_api_token' > /etc/hetzner-dns-token
chmod 600 /etc/hetzner-dns-token
```

The token file path can be overridden with `HETZNER_DNS_TOKEN_FILE=/path/to/token`.

## Usage

**Single domain**:

```bash
certbot certonly -n \
  --agree-tos --no-eff-email \
  -m 'admin@example.org' \
  --manual --preferred-challenges=dns \
  --manual-auth-hook '/usr/local/bin/certbot-hook-hetzner auth' \
  --manual-cleanup-hook '/usr/local/bin/certbot-hook-hetzner cleanup' \
  -d customer.example.org
```

**Wildcard certificate**:

```bash
certbot certonly -n \
  --agree-tos --no-eff-email \
  -m 'admin@example.org' \
  --manual --preferred-challenges=dns \
  --manual-auth-hook '/usr/local/bin/certbot-hook-hetzner auth' \
  --manual-cleanup-hook '/usr/local/bin/certbot-hook-hetzner cleanup' \
  -d example.org -d '*.example.org'
```

## Delegation via `CNAME`

It's possible to delegate the DNS validation of a domain A to another domain B. Common example: A is a customer and B is a hosting provider.

Advantages:

- a domain can be validated without the host needing to be reachable via HTTP from the internet (like for internal systems behind a firewall)
- a (customer) domain can be validated without the customer needing a secret token
- a (customer) domain can be validated without the hosting provider needing permissions to configure the customer domain's DNS
- because it is a DNS challenge, a _wildcard_ certificate can also be issued

### How it works

Create a CNAME record in the customer's DNS that points the ACME challenge to a domain you control:

```text
_acme-challenge.customer.example.org.  IN  CNAME  _acme-challenge.customer._validation.example.com.
```

The hook script automatically detects the CNAME and creates the TXT record in the target zone instead.

### Further reading

- [Let's Encrypt: Onboarding Your Customers](https://letsencrypt.org/2019/10/09/onboarding-your-customers-with-lets-encrypt-and-acme.html)
- [Hetzner Community: Let's Encrypt DNS](https://community.hetzner.com/tutorials/letsencrypt-dns)

## Upgrading from v1

v1 used the legacy Hetzner DNS Console API (`dns.hetzner.com/api/v1`), which is being retired in May 2026. v2 uses the new Hetzner Cloud API (`api.hetzner.cloud/v1`).

**What you need to do:**

1. **Get a new API token** from [console.hetzner.com](https://console.hetzner.com) > Security > API Tokens (Read & Write). The old DNS Console tokens from `dns.hetzner.com` will not work.

2. **Migrate your DNS zones** to the new Hetzner Console if you haven't already. See [Hetzner's migration guide](https://docs.hetzner.com/networking/dns/migration-to-hetzner-console/process).

3. **Update the token file**:
   ```bash
   echo 'your_new_cloud_api_token' > /etc/hetzner-dns-token
   ```

4. **Update the script**:
   ```bash
   curl -sL "https://github.com/functional-swiss/certbot-dns-hetzner/raw/v2-console-api/certbot-hook-hetzner" \
     > /usr/local/bin/certbot-hook-hetzner && chmod +x /usr/local/bin/certbot-hook-hetzner
   ```

5. **Test** with `--staging` before using in production:
   ```bash
   certbot certonly -n --staging \
     --agree-tos --no-eff-email -m 'admin@example.org' \
     --manual --preferred-challenges=dns \
     --manual-auth-hook '/usr/local/bin/certbot-hook-hetzner auth' \
     --manual-cleanup-hook '/usr/local/bin/certbot-hook-hetzner cleanup' \
     -d test.yourdomain.com
   ```

## Fork

The repo was forked to fix:

- Hetzner script tries the real domain, not the delegated one (solved by looking up CNAME via `dig`)
- Original project [was declared unmaintained](https://github.com/dschoeffm/hetzner-dns-certbot/issues/9#issuecomment-949838474)
- v2: Migrated to Hetzner Cloud API before DNS Console retirement
