# Development Guide

## API Reference: Old vs New

The script was migrated from the legacy Hetzner DNS Console API to the Hetzner Cloud API in v2.

| Aspect | Old (v1) `dns.hetzner.com/api/v1` | New (v2) `api.hetzner.cloud/v1` |
|--------|----------------------------------|---------------------------------|
| Auth header | `Auth-API-Token: TOKEN` | `Authorization: Bearer TOKEN` |
| Zone lookup | `GET /zones?search_name=X` | `GET /zones?name=X` |
| Zone ID type | String UUID | Numeric ID |
| Create TXT | `POST /records` with full record body | `POST /zones/{zid}/rrsets/{name}/TXT/actions/add_records` |
| Delete TXT | List records + `DELETE /records/{rid}` | `POST /zones/{zid}/rrsets/{name}/TXT/actions/remove_records` |
| Record name | FQDN with trailing dot (`_acme-challenge.test.kultr.de.`) | Relative to zone (`_acme-challenge.test`) |
| TXT values | Plain: `myvalue` | Quoted: `"\"myvalue\""` |
| Minimum TTL | 1 second | 60 seconds |
| Async actions | None | Poll `GET /actions/{id}` until `status: "success"` |
| Action response | N/A | `{ "action": {...} }` (singular, not array!) |
| Token source | dns.hetzner.com console | console.hetzner.com > Security > API Tokens |

### Curl Examples

**Old API — list zones:**
```bash
curl -H "Auth-API-Token: $TOKEN" \
  "https://dns.hetzner.com/api/v1/zones?search_name=kultr.de"
```

**New API — list zones:**
```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.hetzner.cloud/v1/zones?name=kultr.de"
```

**New API — create TXT record:**
```bash
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.hetzner.cloud/v1/zones/${ZONE_ID}/rrsets/_acme-test/TXT/actions/add_records" \
  -d '{"ttl": 60, "records": [{"value": "\"test-value-123\""}]}'
```

**New API — remove TXT record:**
```bash
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.hetzner.cloud/v1/zones/${ZONE_ID}/rrsets/_acme-test/TXT/actions/remove_records" \
  -d '{"records": [{"value": "\"test-value-123\""}]}'
```

## Authentication Setup

1. Go to [console.hetzner.com](https://console.hetzner.com)
2. Navigate to **Security** > **API Tokens**
3. Click **Generate API Token**
4. Give it a name (e.g. `certbot-dns`) and select **Read & Write** permissions
5. Copy the token and save it to `/etc/hetzner-dns-token` on the server

**Important:** This is a _Cloud API_ token, not the old DNS Console token from `dns.hetzner.com`. The old tokens will stop working when the DNS Console is retired (May 2026).

## Token Location

The script reads the token from `$HETZNER_DNS_TOKEN_FILE` (default: `/etc/hetzner-dns-token`).

For local development/testing, you can override this:
```bash
export HETZNER_DNS_TOKEN_FILE=/tmp/hetzner-dns-token
```

## Manual Curl Tests (against `kultr.de`)

Set up:
```bash
export TOKEN=$(cat "${HETZNER_DNS_TOKEN_FILE:-/etc/hetzner-dns-token}")
export API="https://api.hetzner.cloud/v1"
```

### Test 1: List zones / verify API access

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API/zones" | jq '.zones[] | {id, name}'
```

Expected: list of zones including `kultr.de`.

### Test 2: Look up `kultr.de` zone ID

```bash
ZONE_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "$API/zones?name=kultr.de" | jq -r '.zones[] | select(.name == "kultr.de") | .id')
echo "Zone ID: $ZONE_ID"
```

### Test 3: Create test TXT record

```bash
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API/zones/${ZONE_ID}/rrsets/_acme-test/TXT/actions/add_records" \
  -d '{"ttl": 60, "records": [{"value": "\"hello-from-dev-test\""}]}' | jq .
```

### Test 4: Verify via DNS

```bash
# may take 30-60s to propagate
dig @8.8.8.8 -t TXT _acme-test.kultr.de +short
```

Expected: `"hello-from-dev-test"`

### Test 5: Remove the test record

```bash
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API/zones/${ZONE_ID}/rrsets/_acme-test/TXT/actions/remove_records" \
  -d '{"records": [{"value": "\"hello-from-dev-test\""}]}' | jq .
```

### Test 6: Verify removal

```bash
dig @8.8.8.8 -t TXT _acme-test.kultr.de +short
# should return empty after propagation
```

## Script Unit Tests (env var simulation)

These simulate what certbot does when it calls the hook:

### Test auth (create record)

```bash
TIMESTAMP=$(date +%s)
HETZNER_DNS_TOKEN_FILE=/tmp/hetzner-dns-token \
CERTBOT_DOMAIN=kultr.de \
CERTBOT_VALIDATION="test-${TIMESTAMP}" \
  ./certbot-hook-hetzner auth
```

Verify: `dig @8.8.8.8 -t TXT _acme-challenge.kultr.de +short` should show the value.

### Test cleanup (remove record)

```bash
HETZNER_DNS_TOKEN_FILE=/tmp/hetzner-dns-token \
CERTBOT_DOMAIN=kultr.de \
CERTBOT_VALIDATION="test-${TIMESTAMP}" \
  ./certbot-hook-hetzner cleanup
```

Verify: `dig @8.8.8.8 -t TXT _acme-challenge.kultr.de +short` should be empty.

### Test with subdomain

```bash
TIMESTAMP=$(date +%s)
HETZNER_DNS_TOKEN_FILE=/tmp/hetzner-dns-token \
CERTBOT_DOMAIN=sub.kultr.de \
CERTBOT_VALIDATION="test-sub-${TIMESTAMP}" \
  ./certbot-hook-hetzner auth

# record should be _acme-challenge.sub in the kultr.de zone
dig @8.8.8.8 -t TXT _acme-challenge.sub.kultr.de +short

HETZNER_DNS_TOKEN_FILE=/tmp/hetzner-dns-token \
CERTBOT_DOMAIN=sub.kultr.de \
CERTBOT_VALIDATION="test-sub-${TIMESTAMP}" \
  ./certbot-hook-hetzner cleanup
```

## Integration Tests (certbot --staging)

All tests use `--staging` to avoid Let's Encrypt rate limits.

### Test 1: Simple domain (direct zone)

```bash
certbot certonly -n --staging \
  --agree-tos --no-eff-email -m 'ops@functional.swiss' \
  --manual --preferred-challenges=dns \
  --manual-auth-hook './certbot-hook-hetzner auth' \
  --manual-cleanup-hook './certbot-hook-hetzner cleanup' \
  -d test.kultr.de
```

### Test 2: Wildcard certificate

```bash
certbot certonly -n --staging \
  --agree-tos --no-eff-email -m 'ops@functional.swiss' \
  --manual --preferred-challenges=dns \
  --manual-auth-hook './certbot-hook-hetzner auth' \
  --manual-cleanup-hook './certbot-hook-hetzner cleanup' \
  -d letsencrypt-test.kiste.li -d '*.letsencrypt-test.kiste.li'
```

### Test 3: CNAME delegation (full feature test)

**Prerequisite**: Create CNAME in Gandi DNS for `kiste.li`:

```
_acme-challenge.letsencrypt-test.kiste.li  CNAME  _acme-challenge.letsencrypt-test.kultr.de
```

Then the hook resolves the CNAME and creates the TXT record in the `kultr.de` zone:

```bash
certbot certonly -n --staging \
  --agree-tos --no-eff-email -m 'ops@functional.swiss' \
  --manual --preferred-challenges=dns \
  --manual-auth-hook './certbot-hook-hetzner auth' \
  --manual-cleanup-hook './certbot-hook-hetzner cleanup' \
  -d letsencrypt-test.kiste.li
```

## Code Quality

```bash
shellcheck certbot-hook-hetzner
```

Should pass with zero warnings.

## Reference Implementations

- [acme.sh dns_hetznercloud.sh](https://github.com/acmesh-official/acme.sh/blob/master/dnsapi/dns_hetznercloud.sh) — mature implementation of the new API
- [Hetzner Cloud API docs](https://docs.hetzner.cloud/) — official API reference
