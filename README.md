<p align="center"><img src="assets/logo.png" alt="Pro Gineous" width="360"></p>

# dns-email-records

Which DNS records a domain needs so its mail is delivered and cannot be spoofed easily, with zone file examples for Google Workspace, Microsoft 365, Zoho Mail and a self-hosted server.

Provider-specific values come from each provider's documentation, linked next to the record. Where a value is unique to your account (DKIM keys, Microsoft's MX host), the example says `REPLACE` and tells you where to copy it from. Do not guess those.

## Files

```
zones/google-workspace.zone   BIND-format examples, one per provider
zones/microsoft-365.zone
zones/zoho-mail.zone
zones/self-hosted.zone
mta-sts/<provider>.txt        matching MTA-STS policy files, in testing mode
```

Most DNS panels do not take zone files. Read the zone file for the host name, type and value of each record and type them into the panel. A host of `@` means the bare domain; `_dmarc` means `_dmarc.example.com`.

## The records, in the order to add them

### 1. MX

Tells other servers where to deliver mail for your domain. Lower preference number wins. Point it at exactly what your provider says; remove the old provider's MX records when you switch, or mail will split between the two.

### 2. SPF (TXT on the bare domain)

Lists who may send mail with your domain in the envelope sender. Rules that cause most of the problems:

- One SPF record per name. Two `v=spf1` TXT records is a permerror and SPF fails for everything.
- At most 10 DNS lookups (`include`, `a`, `mx`, `redirect`, `exists`). Every newsletter tool or CRM you add an `include` for counts against that.
- Combine providers in one record: `v=spf1 include:_spf.google.com include:servers.mcsv.net ~all`.

Spec: [RFC 7208](https://www.rfc-editor.org/rfc/rfc7208).

### 3. DKIM

The provider signs outgoing mail with a private key; you publish the public key under `<selector>._domainkey`. Google and Zoho give you a TXT record. Microsoft 365 gives you two CNAMEs that point to keys it hosts and rotates.

A 2048-bit key does not fit in one 255-character TXT string. In a zone file, split it into several quoted strings inside one record (the examples show how). Most web panels do the split for you if you paste the whole key.

Spec: [RFC 6376](https://www.rfc-editor.org/rfc/rfc6376).

### 4. DMARC (TXT at `_dmarc`)

Ties SPF and DKIM to the visible From address and tells receivers what to do when both fail. Start with:

```
v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com
```

Read the aggregate reports for a few weeks, fix every legitimate sender that fails, then move to `p=quarantine` and later `p=reject`. Gmail and Yahoo require a DMARC record (even `p=none`) from bulk senders.

If `rua` points to a mailbox on another domain, that domain has to publish an authorization record (`example.com._report._dmarc.otherdomain.com TXT "v=DMARC1"`), or reports will not be sent there.

Spec: [RFC 7489](https://www.rfc-editor.org/rfc/rfc7489).

### 5. TLS-RPT (TXT at `_smtp._tls`)

Asks other mail servers to send you daily reports about TLS problems when delivering to you. Harmless, and useful before you turn on MTA-STS.

```
_smtp._tls  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

Spec: [RFC 8460](https://www.rfc-editor.org/rfc/rfc8460).

### 6. MTA-STS

Tells senders your MX hosts support TLS with valid certificates and that they should refuse to deliver over a downgraded connection. Two parts:

1. A policy file served over HTTPS at `https://mta-sts.example.com/.well-known/mta-sts.txt` with a valid certificate for `mta-sts.example.com`. See [mta-sts/](mta-sts/).
2. A TXT record at `_mta-sts` with an `id` that you change every time you edit the policy.

Every `mx:` line must match an MX host. Start with `mode: testing`, watch the TLS-RPT reports for a week or two, then switch to `enforce` and change the `id`.

The file is plain text; any web server works. With Caddy:

```
mta-sts.example.com {
	root * /var/www/mta-sts
	file_server
}
```

with the policy at `/var/www/mta-sts/.well-known/mta-sts.txt`.

Before you enforce, check that each MX host presents a certificate that matches its name:

```bash
openssl s_client -connect mx.example.net:25 -starttls smtp -servername mx.example.net </dev/null 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName
```

Spec: [RFC 8461](https://www.rfc-editor.org/rfc/rfc8461).

### 7. CAA

Not a mail record, but it belongs on every domain: it limits which certificate authorities may issue certificates for it, including the `mta-sts` host. The examples allow Let's Encrypt only and forbid wildcards. Add a line for each CA you actually use.

Spec: [RFC 8659](https://www.rfc-editor.org/rfc/rfc8659). Let's Encrypt's identifier: [letsencrypt.org/docs/caa](https://letsencrypt.org/docs/caa/).

## Provider notes

### Google Workspace

- MX is a single record, `1 smtp.google.com`. Older setups use five `ASPMX` records; Google says those keep working. If you use the old set, the MTA-STS policy must list those hosts instead.
- SPF `include:_spf.google.com`. DKIM selector defaults to `google`.
- Google's admin console can check your MTA-STS setup and suggest a policy.

Docs: [MX](https://support.google.com/a/answer/174125), [SPF](https://support.google.com/a/answer/10685031), [DKIM](https://support.google.com/a/answer/174124), [DMARC](https://support.google.com/a/answer/2466580), [MTA-STS policy](https://support.google.com/a/answer/9276511), [MTA-STS and TLS-RPT records](https://support.google.com/a/answer/9276512), [all TXT values](https://support.google.com/a/answer/2716802).

### Microsoft 365

- The MX host is specific to your domain and shown in the admin center. Copy it.
- SPF `include:spf.protection.outlook.com` with `-all`, as in Microsoft's docs.
- DKIM is two CNAMEs, `selector1` and `selector2`. Publish both, then enable signing in the Defender portal.
- Microsoft documents `mx: *.mail.protection.outlook.com` for MTA-STS when your MX points straight at Exchange Online, and notes the wildcard is specific to Exchange Online.

Docs: [DNS records for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/admin/get-help-with-domains/create-dns-records-at-any-dns-hosting-provider), [SPF](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-spf-configure), [DKIM](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-dkim-configure), [DMARC](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-dmarc-configure), [MTA-STS](https://learn.microsoft.com/en-us/purview/enhancing-mail-flow-with-mta-sts).

### Zoho Mail

- Three MX records, preference 10, 20 and 50. The example uses the US data center (`zoho.com`); accounts in other regions use a different TLD. Use what your admin console shows.
- SPF `include:zohomail.com` for the US data center.
- You pick the DKIM selector name yourself when generating the key.
- Zoho does not publish an MTA-STS policy for customers to copy. The `mta-sts/zoho-mail.txt` file lists the MX hosts from the example; run the `openssl` check above against each one before going past `testing`.

Docs: [MX](https://www.zoho.com/mail/help/adminconsole/configure-email-delivery.html), [SPF](https://www.zoho.com/mail/help/adminconsole/spf-configuration.html), [DKIM](https://www.zoho.com/mail/help/adminconsole/dkim-configuration.html).

### Self-hosted

- Set reverse DNS (PTR) for the mail server's IP to the mail host name. You do this at your VPS provider, not in your zone.
- Check your provider allows outbound port 25. Many block it on new accounts until you ask.
- `v=spf1 mx -all` covers a server that both receives and sends. Add `ip4:`/`ip6:` for any other sending host.
- The DKIM key comes from your mail software (`opendkim-genkey`, rspamd, or the admin UI of mailcow, Stalwart and similar).
- The SRV records help mail clients find your IMAP and submission ports ([RFC 6186](https://www.rfc-editor.org/rfc/rfc6186), [RFC 8314](https://www.rfc-editor.org/rfc/rfc8314)).

## Checking what is published

```bash
dig +short MX example.com
dig +short TXT example.com
dig +short TXT google._domainkey.example.com
dig +short TXT _dmarc.example.com
dig +short TXT _smtp._tls.example.com
dig +short TXT _mta-sts.example.com
dig +short CAA example.com
curl -s https://mta-sts.example.com/.well-known/mta-sts.txt
```

The [server-toolkit](https://gitlab.com/pro-gineous-group/server-toolkit) has `email/mail-auth-check.sh`, which runs the SPF, DKIM and DMARC lookups with a few sanity checks.

## Validating the examples

Each zone file loads cleanly:

```bash
for z in zones/*.zone; do named-checkzone example.com "$z"; done
```

`named-checkzone` comes with `bind9-utils` on Debian/Ubuntu. It checks syntax, not whether the values are right for your account.

## License

MIT, see [LICENSE](LICENSE).

---

Written by [Pro Gineous](https://progineous.com/), where domains, DNS and hosting are the day job.
