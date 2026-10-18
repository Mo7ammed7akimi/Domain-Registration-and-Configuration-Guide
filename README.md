# Domain Registration and Configuration Guide

A step-by-step guide to registering a new domain and setting up the configuration that matters most: security, DNS, email and HTTPS.

## Contents

1. [Choose a domain name](#1-choose-a-domain-name)
2. [Check availability](#2-check-availability)
3. [Choose a registrar](#3-choose-a-registrar)
4. [Register the domain](#4-register-the-domain)
5. [Secure the account and the domain](#5-secure-the-account-and-the-domain)
6. [Configure DNS](#6-configure-dns)
7. [Set up email authentication](#7-set-up-email-authentication)
8. [Enable DNSSEC](#8-enable-dnssec)
9. [Enable HTTPS](#9-enable-https)
10. [Verify the setup](#10-verify-the-setup)
11. [Ongoing maintenance](#11-ongoing-maintenance)
12. [Quick checklist](#quick-checklist)

---

## 1. Choose a domain name

- Keep it short, easy to spell and easy to say aloud.
- Avoid hyphens, numbers and ambiguous spellings.
- Choose a top-level domain (TLD) that fits the purpose:
  - `.com`, `.net`, `.org`: general use
  - `.edu`, `.ac.*`: academic institutions (restricted)
  - Country codes such as `.de`, `.uk`: regional presence (some require local residency)
  - `.dev`, `.io`, `.me`: developers and personal sites
- Check that the name does not infringe an existing trademark.
- Check that the matching social media and GitHub handles are available.

## 2. Check availability

- Search on the registrar's website, or use a WHOIS/RDAP lookup.
- If the name is taken, try another TLD or a small variation.

## 3. Choose a registrar

Compare registrars on:

| Factor | What to look for |
|---|---|
| Price | Both first-year and renewal price (renewal is often higher) |
| ICANN accreditation | The registrar should be accredited |
| WHOIS privacy | Free redaction of personal data |
| DNS management | A clear interface, API access, support for all record types |
| Security | Two-factor authentication, registrar lock, DNSSEC support |
| Transfer policy | No unreasonable fees to leave |
| Support | Reachable when something goes wrong |

## 4. Register the domain

1. Create an account with the registrar.
2. Add the domain to the cart and choose the registration period (1 to 10 years).
3. Enter accurate registrant details. ICANN requires accurate contact data, and a verification email is sent to the registrant address.
4. **Verify the registrant email within the deadline** (usually 15 days). Unverified domains can be suspended.
5. Enable WHOIS privacy if it is not on by default.
6. Enable auto-renew.
7. Complete payment and keep the receipt.

## 5. Secure the account and the domain

Do this immediately after registration.

- **Use a strong, unique password** and a password manager.
- **Enable two-factor authentication** (an authenticator app or hardware key is better than SMS).
- **Use a dedicated email address** for the registrar account, protected by its own 2FA.
- **Enable the registrar lock** (clientTransferProhibited) to prevent unauthorised transfers.
- **Keep contact details current** so renewal and abuse notices reach you.
- **Keep auto-renew on** and a valid payment method on file.
- Store the transfer (EPP/auth) code securely and do not share it.

## 6. Configure DNS

By default the registrar's nameservers are used. You can instead delegate to a DNS provider (for example Cloudflare or a cloud provider) by changing the nameserver (NS) records at the registrar.

### Common record types

| Type | Purpose | Example |
|---|---|---|
| `A` | Maps a name to an IPv4 address | `@ → 203.0.113.10` |
| `AAAA` | Maps a name to an IPv6 address | `@ → 2001:db8::10` |
| `CNAME` | Alias to another hostname | `www → example.com` |
| `MX` | Mail servers (with priority) | `@ → 10 mail.example.com` |
| `TXT` | Free text (SPF, DKIM, DMARC, verification) | `@ → "v=spf1 ..."` |
| `NS` | Authoritative nameservers | `@ → ns1.provider.com` |
| `CAA` | Which certificate authorities may issue certificates | `@ → 0 issue "letsencrypt.org"` |

### Key points

- A `CNAME` cannot be set on the root (apex) domain `@`. Use `A`/`AAAA` records, or `ALIAS`/`ANAME` if the provider supports them.
- Set `www` as a `CNAME` to the apex, or the reverse, and redirect one to the other so there is a single canonical address.
- Use a low TTL (for example 300 seconds) while setting up, then raise it (for example 3600 or higher) once stable.
- Add a `CAA` record to restrict certificate issuance to the authority you use.
- Remove unused records, especially those pointing to services you no longer use (to avoid subdomain takeover).

## 7. Set up email authentication

If the domain will send or receive email, configure these three `TXT` records. They protect the domain from spoofing and improve deliverability.

**SPF** lists the servers allowed to send mail for the domain:

```
Type: TXT   Name: @
Value: v=spf1 include:_spf.your-mail-provider.com ~all
```

Only one SPF record is allowed per domain.

**DKIM** adds a cryptographic signature to outgoing mail. The mail provider supplies the selector and public key:

```
Type: TXT   Name: selector._domainkey
Value: v=DKIM1; k=rsa; p=<public-key>
```

**DMARC** tells receivers what to do when SPF/DKIM checks fail:

```
Type: TXT   Name: _dmarc
Value: v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com
```

Start with `p=none` to monitor, then move to `p=quarantine` and finally `p=reject` once legitimate mail passes consistently.

If the domain will **not** send mail, publish a null setup to prevent abuse:

```
@        TXT  "v=spf1 -all"
_dmarc   TXT  "v=DMARC1; p=reject;"
```

## 8. Enable DNSSEC

DNSSEC signs DNS responses so they cannot be forged in transit.

1. Enable DNSSEC at the DNS provider. It generates a DS record.
2. Add the DS record at the registrar (if the registrar and DNS provider are different).
3. Verify with `dig +dnssec example.com`.

If the DS record is wrong or left behind after changing DNS provider, the domain can stop resolving. Remove the DS record before moving DNS.

## 9. Enable HTTPS

- Obtain a TLS certificate from a certificate authority. Let's Encrypt is free and automated.
- Most hosting platforms issue and renew certificates automatically once DNS points to them.
- Redirect all HTTP traffic to HTTPS.
- Consider enabling HSTS once HTTPS works reliably on all subdomains.

## 10. Verify the setup

Propagation can take from a few minutes up to 48 hours.

Run these in a terminal. `nslookup` is built into Windows, macOS and Linux. Replace `example.com` with your domain.

```bash
# Nameservers
nslookup -type=NS example.com

# A / AAAA records
nslookup -type=A example.com
nslookup -type=AAAA example.com

# CNAME
nslookup -type=CNAME www.example.com

# Mail records
nslookup -type=MX example.com
nslookup -type=TXT example.com
nslookup -type=TXT _dmarc.example.com
```

To query a specific DNS server instead of the default one, add its address at the end:

```bash
nslookup -type=NS example.com 8.8.8.8
```

Other checks:

```bash
# DNSSEC (nslookup cannot show this; use dig, WSL or an online tool such as DNSViz)
dig +dnssec example.com

# HTTPS and redirects (on Windows PowerShell, use curl.exe)
curl -I https://example.com

# Registration details (on Windows, use an online WHOIS/RDAP lookup)
whois example.com
```

Online tools such as DNS propagation checkers, SSL Labs and MXToolbox are useful for a second opinion.

## 11. Ongoing maintenance

- Renew before expiry. Expired domains can be registered by others after the grace period.
- Review DNS records every few months and delete anything unused.
- Review DMARC reports and tighten the policy over time.
- Keep registrant contact details and 2FA recovery methods current.
- Keep the transfer lock on unless you are deliberately transferring.

### Transferring to another registrar

1. Unlock the domain and request the auth (EPP) code.
2. Ensure the domain is older than 60 days and was not transferred in the last 60 days.
3. Start the transfer at the new registrar and approve the confirmation email.
4. Keep DNS records identical during the move to avoid downtime.

## Quick checklist

- [ ] Domain name chosen and checked
- [ ] Registered with an accredited registrar
- [ ] Registrant email verified
- [ ] WHOIS privacy enabled
- [ ] Auto-renew enabled
- [ ] 2FA enabled on the registrar account
- [ ] Registrar lock enabled
- [ ] Nameservers set
- [ ] A/AAAA/CNAME records created
- [ ] CAA record added
- [ ] SPF, DKIM and DMARC configured
- [ ] DNSSEC enabled
- [ ] HTTPS enforced
- [ ] Setup tested with `dig` and `curl`

## Licence

This guide is released under the MIT Licence. Contributions and corrections are welcome through issues and pull requests.
