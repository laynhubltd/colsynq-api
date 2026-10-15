# v4 research: tenant resolution by subdomain, custom domain and X-Tenant

Status: research notes for IAM/Tenancy v4 · Date: 2026-10-13 · Reviews: `iam_v3_1_snapshot.md` sections 4.1, 7, 8, 9.2, 10
Stack: Symfony 8.1, API Platform 5, DBAL 4, PostgreSQL, FrankenPHP (Caddy inside), database per tenant.

Legend: [S] = stated by the cited source. [I] = my inference or recommendation.

---

## 0. Summary of gaps in v3.1

| # | v3.1 today | Gap | Recommended change |
|---|---|---|---|
| 1 | `tenant_domain(host PK, tenant_id, is_primary, verification_token_hash, verified_at, created_at)` | No status, no re-check, no removal, no cert state, no method, no history; hashing a token that is published in public DNS adds nothing | New `tenant_domain` shape (section 9) with `id`, `status`, `verification_method`, plain `verification_token`, `last_checked_at`, `check_failures`, `cert_status`, `cert_expires_at`, `removed_at`; partial unique on `host` |
| 2 | Verify once (`verified_at`) | Ownership is never checked again; a domain sold or moved stays mapped to the tenant and keeps getting certificates | Daily re-check of the TXT record and the routing record; 7-day grace, then `inactive`; stop cert renewal |
| 3 | No TLS design | Unknown: wildcard, on-demand, rate limits, CAA, monitoring | Wildcard `*.colsynq.app` via DNS-01; Caddy (FrankenPHP) on-demand TLS for custom domains only, gated by an `ask` endpoint reading `tenant_domain`; or Cloudflare for SaaS |
| 4 | "Header and host values are normalised (lower case, no port)" | No trailing dot, IDN/punycode, multi-level subdomain, reserved host classes, SNI/Host mismatch, trusted_hosts with dynamic custom domains | Host classifier (section 3.3) and `strict_sni_host`; X-Forwarded-Host not trusted |
| 5 | Reserved slugs list in 5.1 | No homoglyph/confusable check, no quarantine of old slugs, no slug history | `tenant_slug_history` table; reserved list in code; slug ASCII only already; add near-duplicate check (edit distance / skeleton) and squatting review |
| 6 | Slug can change, "nothing keyed on it" | Old links, emails, bookmarks and mobile clients with stored X-Tenant break; old slug can be taken by another society (link hijack) | Keep old slug as a redirecting alias 90 days and never reassign (or quarantine 12 months) |
| 7 | `__Host-` session cookie, SameSite=Lax (10) | All `<slug>.colsynq.app` hosts are the *same site*: SameSite does not stop tenant A's subdomain from sending requests to tenant B; cookie tossing for non-prefixed cookies | Keep `__Host-` for every cookie including CSRF; add Fetch Metadata rule: reject unsafe methods unless `Sec-Fetch-Site: same-origin`; serve tenant-uploaded files (logo_url, SVG) from a separate sandbox domain |
| 8 | Handoff code (9.2) | Not bound to target host; passkeys (WebAuthn RP ID) cannot work on custom domains | Central `auth.colsynq.app` for all credential ceremonies (password, passkey, OIDC); handoff code bound to session + target host + PKCE-style verifier, redeemed back-channel by the target BFF |
| 9 | 60 s cache incl. misses | Negative cache on raw Host strings can be flooded; invalidation across APCu on many nodes is not "at once" | Classify host before lookup; negative-cache only valid-looking names; Redis (shared) for invalidation or pub/sub to clear APCu; versioned cache keys |
| 10 | 423 for suspended | 423 (WebDAV "Locked") is unusual; also leaks existence | Keep 423 if wanted but return RFC 9457 problem+json `type=tenant-suspended`; decide enumeration policy explicitly (section 7) |
| 11 | Email links | Not specified which host links use | Links built from `TenantDirectory::canonicalUrl(tenantId)`: active primary custom domain, else `https://<slug>.colsynq.app`; never from the request Host; `framework.router.default_uri` for workers |
| 12 | X-Tenant vs host conflict → 400 | No rule for which hosts accept X-Tenant; custom header forces CORS preflight; caches must vary | Shared hosts (`api.`, `app.`) require X-Tenant; tenant hosts make X-Tenant optional and must match; `Vary: X-Tenant, Origin`; tenant data `Cache-Control: private, no-store` |
| 13 | HSTS not specified | `.app` is preloaded at TLD level; custom domains must NOT get `includeSubDomains` from us | HSTS `max-age=31536000; includeSubDomains` on colsynq.app hosts; plain `max-age` (no includeSubDomains, no preload) on customer domains |

---

## 1. Custom domain onboarding

### 1.1 Verification methods

- IETF draft *Domain Control Validation using DNS* (draft-ietf-dnsop-domain-verification-techniques-13, June 2026, BCP track) [S]:
  - "The RECOMMENDED method of doing DNS-based domain control validation is to use DNS TXT records."
  - Owner name: application-specific underscore label, e.g. `_example_service-challenge.example.com`.
  - Token: unique, adequate entropy (RFC 4086); base32 or hex RECOMMENDED when inside a DNS label; base64url allowed in RDATA.
  - Persistent validation is only appropriate when the provider keeps the validation bound to a specific user account; providers MUST explain how revocation and re-validation work; MAY monitor and request re-confirmation.
  - Delegated (CNAME) validation: intermediary MUST keep the association of user/domain/token to prevent dangling-CNAME takeovers.
  - Validation records SHOULD have short TTL; providers MUST use a DNSSEC-validating resolver; ICANN public suffixes SHOULD NOT be validated.
  - Source: https://datatracker.ietf.org/doc/draft-ietf-dnsop-domain-verification-techniques/
- Cloudflare for SaaS [S]: ownership pre-validation via TXT `_cf-custom-hostname.<host>` = UUID, or HTTP `http://<host>/.well-known/cf-custom-hostname-challenge/<id>`; or real-time validation when the customer's CNAME already points at the SaaS target. Pre-validation avoids downtime; real-time is easiest. Production needs ownership `active`, `ssl.status active` and DNS pointing at the target.
  - https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/domain-support/hostname-validation/
  - https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/domain-support/hostname-validation/pre-validation/
  - https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/domain-support/hostname-validation/realtime-validation/
- GitHub Pages [S]: TXT `_github-pages-challenge-<owner>.<domain>`; keep the record so the domain stays verified; verification prevents takeover when a repo is deleted or Pages disabled; a verified domain "is released" from another account's site when you verify it. Strongly advises against wildcard DNS records because `b.a.example.com` can still be taken over.
  - https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages
- GitLab Pages [S, via handbook search result]: periodic re-verification; on failure the domain is unverified but enabled for a 7-day grace, then expires and is disabled. https://handbook.gitlab.com/handbook/support/workflows/verify_pages_domain/
- Vercel [S]: apex via A record (e.g. 76.76.21.21), subdomain via per-project CNAME; TXT verification required when the domain is used by another account; recommends adding both apex and www and redirecting one to the other. https://vercel.com/docs/domains/working-with-domains/add-a-domain

### 1.2 Recommended Colsynq flow [I]

1. Society admin (permission `domains:manage`, or platform-only at first via `/platform/tenants/{id}/domains`) submits `portal.mysociety.ng`.
2. Normalise (section 3.3). Reject: IP literals, `*.colsynq.app` and any name under our own zones, a bare public suffix (`com.ng`, `org.ng` — check against the PSL), more than 253 chars, labels > 63, wildcards.
3. Create `tenant_domain` row `status = pending`, `verification_token` = 26 chars base32 of 128 random bits (`colsynq-verify=<token>`), `expires_at` = now + 7 days for pending.
4. Customer adds two records:
   - Ownership: `TXT _colsynq-challenge.portal.mysociety.ng "colsynq-verify=<token>"` (keep it; persistent validation bound to the tenant).
   - Routing: `CNAME portal.mysociety.ng → domains.colsynq.app` (subdomain) or `A/AAAA` to our static anycast IPs (apex).
5. Verifier job (Messenger, not the web request) queries authoritative DNS via a DNSSEC-validating resolver, checks both; sets `verified_at`, `status = active`, triggers cert issuance (or creates the Cloudflare custom hostname). HTTP check as fallback: `/.well-known/colsynq-challenge/<id>` only when routing already points to us.
6. Both apex and `www` can be registered; the non-primary one has `redirect_to_primary = true` (308 to the primary). Recommend societies use a subdomain (`portal.`, `app.`) because CNAME at apex is not allowed by DNS; ALIAS/ANAME/CNAME-flattening is provider-specific.
7. Conflict: host already active for tenant B → tenant A may claim it only by proving the TXT record with A's token; on success B's row is `removed` with reason `claimed_by_other_tenant`, B's admins are notified, audit event written (GitHub model).

### 1.3 Re-verification and removal [I, based on draft + GitLab + Cloudflare]

- Daily re-check (jittered) of TXT and routing. Failure → `check_failures + 1`, `status = failing`; after 7 days → `status = inactive`: resolver stops mapping it, ask endpoint says no, certificate not renewed. Owner emailed at first failure and at deactivation.
- Removal by admin → `status = removed`, `removed_at`; cache purged; certificate deleted from storage (Caddy) or custom hostname deleted (Cloudflare API DELETE also deletes certs [S]). Cloudflare note [S]: a churned customer whose hostname stays in your zone may keep routing to you — always delete.
- Cloudflare [S]: when a Cloudflare-hosted customer points away, hostname goes `moved`, then auto-deleted after 7 days.
- Tenant archived → all its domains `inactive`.

### 1.4 Dangling CNAME and takeover

- MDN [S]: takeover happens when DNS points at a provider but no active host claims the name; attacker claims it on the same provider. Owners: create the virtual host first, DNS last; remove DNS first on deprovision. Providers: verify the claimant has a legitimate claim. https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/Subdomain_takeover
- Colsynq risks [I]:
  - Inside our platform: a customer CNAME left pointing at `domains.colsynq.app` after removal could be claimed by another society. Mitigated only because claim requires the TXT proof — never activate on routing alone ("real-time validation" style) unless the hostname is also bound to the tenant (e.g. per-tenant CNAME target `t-<id>.domains.colsynq.app`).
  - Our own side: never release the static IPs or the `domains.colsynq.app` target while any customer may still point at it; releasing an elastic IP / LB hostname creates takeover risk for customers' dangling records. Keep an inventory of our DNS records (no dangling CNAMEs to third-party SaaS such as status pages, help desks).
  - Our wildcard `*.colsynq.app` must point at our own load balancer only (GitHub's wildcard warning applies when the wildcard targets a shared provider).

---

## 2. TLS at scale

### 2.1 Let's Encrypt limits (page updated Aug 2026) [S] https://letsencrypt.org/docs/rate-limits/

| Limit | Value |
|---|---|
| New registrations per IP | 10 / 3 h |
| New orders per account | 300 / 3 h (override on request) |
| New certificates per registered domain | 50 / 7 days, shared across all accounts (override on request) |
| Duplicate certificates (same identifier set) | 5 / 7 days |
| Authorization failures per identifier per account | 5 / hour |
| Consecutive authorization failures | identifier gets **paused** (pause starts after repeated failures; can last days to years) |
| ARI renewals | exempt from all limits |
| Non-ARI renewals (same identifiers) | exempt from orders and registered-domain limits |

- Lifetimes [S]: opt-in `tlsserver` profile issues 45-day certs since 13 May 2026; default `classic` moves to 64 days on 10 Feb 2027 and 45 days (7-hour authz reuse) on 16 Feb 2028; `shortlived` = 6 days. https://isrg.org/post/from-90-to-45/ , https://community.letsencrypt.org/t/upcoming-let-s-encrypt-profile-changes-on-may-13-20th-and-27th/247049
- ARI is RFC 9773 (June 2025). LE integration guide: check ARI at least twice a day; renew at one-third lifetime left; treat renewal failure as non-fatal with back-off (1 min, 10, 100, then daily); store certs durably and reuse them across frontends; stagger renewals. https://letsencrypt.org/docs/integration-guide/ , https://www.rfc-editor.org/rfc/rfc9773.html
- LE stopped expiry emails on 4 June 2025 [S] → we need our own monitoring. https://letsencrypt.org/2025/06/26/expiration-notification-service-has-ended/

Consequences for Colsynq [I]:
- **Never issue per-slug certs for `<slug>.colsynq.app`** (on-demand would hit 50/week on `colsynq.app`). One wildcard `*.colsynq.app` + `colsynq.app` via DNS-01. Wildcard covers one label only; reject `a.b.colsynq.app`.
- Custom domains are each their own registered domain, so the 50/week limit rarely bites; the 300 orders / 3 h account limit bites during bulk onboarding → queue issuance.
- The pause on consecutive failures means: never keep retrying for a domain that no longer points to us. Daily re-check (1.3) gates renewals.

### 2.2 Caddy / FrankenPHP on-demand TLS [S] https://caddyserver.com/docs/automatic-https , https://caddyserver.com/docs/caddyfile/options

- On-demand TLS "must be both enabled and restricted to prevent abuse".
- `ask`: Caddy calls `<url>?domain=<name>`; 2xx allows issuance; anything else cancels and fails the handshake; should answer "in a few milliseconds, ideally" and avoid DNS or network calls.
- `interval`/`burst` are NOT recommended; remove them.
- Caddy internal limit: 10 attempts per ACME account per 10 s.
- Since 2.10, a wildcard certificate covers matching subdomains automatically.
- `strict_sni_host`: requires Host header = TLS SNI, else 421. Only auto-on with client auth → set it explicitly.
- `trusted_proxies`: none trusted by default.
- FrankenPHP is configured with the Caddyfile, so the same options apply. https://frankenphp.dev/docs/config/

Recommended [I]:
```caddyfile
{
    on_demand_tls {
        ask http://127.0.0.1:8081/internal/tls-ask   # internal-only listener
    }
    servers { strict_sni_host on }
}
*.colsynq.app, colsynq.app {
    tls { dns <provider> {env.DNS_TOKEN} }   # DNS-01 wildcard
}
https:// {
    tls { on_demand }
}
```
- `ask` handler: lowercase name → lookup `tenant_domain` where `status in (active, failing)` and tenant not archived → 200 else 404. Reads cache or a small indexed query only. Not reachable from the internet. Rate-limit per name.
- Cert storage shared across nodes (Caddy storage module on Redis/S3/Postgres) so nodes do not issue duplicates.

### 2.3 Managed alternatives

- Cloudflare for SaaS [S]: 100 custom hostnames included, $0.10 each after, max 50,000 on Free/Pro/Business; apex proxying (static IPs/BYOIP), wildcard custom hostnames and custom metadata are Enterprise only. https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/plans/
- AWS CloudFront SaaS Manager [S]: multi-tenant distribution as a template, a "distribution tenant" per customer, shared ACM cert or CloudFront-managed per-tenant cert (HTTP-validated, auto-renewed, key not exportable); customers CNAME to a connection group endpoint. https://aws.amazon.com/blogs/aws/reduce-your-operational-overhead-today-with-amazon-cloudfront-saas-manager
- AWS ALB alone: 25 certificates per ALB by default (adjustable) → not a fit for per-domain certs at scale. https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html
- Decision [I]: start with FrankenPHP/Caddy on-demand (zero extra vendor, fits the stack); keep `tenant_domain` provider-neutral (`cert_provider` column) so moving to Cloudflare for SaaS later is a data migration, not a redesign. Nigeria latency: a CDN edge in Lagos (Cloudflare has one) is a reason to consider Cloudflare later.

### 2.4 CAA

- RFC 8659 (`issue`, `issuewild`, `iodef`) and RFC 8657 (`accounturi`, `validationmethods`). https://www.rfc-editor.org/rfc/rfc8657.html
- LE supports both; issuer value `letsencrypt.org`; CAA lookup follows CNAMEs, so a customer `CNAME → domains.colsynq.app` is governed by CAA on our target. https://letsencrypt.org/docs/caa/
- Recommended [I] on `colsynq.app`:
  ```
  colsynq.app. CAA 0 issue "letsencrypt.org; accounturi=https://acme-v02.api.letsencrypt.org/acme/acct/<id>; validationmethods=dns-01"
  colsynq.app. CAA 0 issuewild "letsencrypt.org; accounturi=...; validationmethods=dns-01"
  colsynq.app. CAA 0 iodef "mailto:security@colsynq.app"
  ```
  - Caddy by default can fall back to ZeroSSL; either pin LE only in Caddy or add a `sectigo.com` record. Make sure `domains.colsynq.app` (CNAME target) CAA allows `http-01`/`tls-alpn-01` for customer domains, or customer issuance fails. Tell customers with their own CAA to allow `letsencrypt.org` (apex A-record setups do not follow our CAA).

### 2.5 HSTS

- hstspreload.org [S]: preload needs `max-age ≥ 31536000`, `includeSubDomains`, `preload`; applies to all subdomains including internal; "cannot easily be undone". https://hstspreload.org/
- `.app` is preloaded at TLD level by Google, so every `colsynq.app` host is HTTPS-only in browsers already [S, secondary sources]. https://www.thesslstore.com/blog/google-launches-dot-app/
- Recommended [I]: send `Strict-Transport-Security: max-age=31536000; includeSubDomains` on colsynq.app hosts (harmless). On custom domains send `max-age=31536000` only — never `includeSubDomains` or `preload`, because we do not own the customer's other subdomains (e.g. their intranet). LE guide warns HSTS can make a customer's site unavailable if they later move to a host without HTTPS.

### 2.6 Monitoring [I]

- Metrics per host: `cert_expires_at`, last renewal error; alert when < 14 days (45-day certs) or two failed renewals.
- Certificate Transparency monitoring for `colsynq.app` (crt.sh / Cert Spotter / CT monitor service) to detect unexpected issuance.
- Synthetic check of a sample of tenant hosts and all custom domains daily (TLS handshake + expected 200 on `/healthz`).

---

## 3. Host header security

### 3.1 Sources
- PortSwigger [S]: Host attacks = password-reset poisoning, web-cache poisoning, routing-based SSRF, override headers (`X-Forwarded-Host`) often supported by default. Prevention: avoid Host in server code, take absolute URLs from config, allow-list Host, disable override headers, allow-list at the proxy. https://portswigger.net/web-security/host-header
- OWASP Forgot Password [S]: "Don't rely on the Host header while creating the reset URLs"; URL "should either be hard-coded, or validated against a list of trusted domains". https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Symfony `trusted_hosts` [S]: list of regexes; mismatch → 400; default empty = any host; env `SYMFONY_TRUSTED_HOSTS`. https://symfony.com/doc/current/reference/configuration/framework.html
- Symfony proxies [S]: trusting `x-forwarded-host` exposes Host header attacks — enable only if the proxy sends it; block direct access to app servers. https://symfony.com/doc/current/deployment/proxies.html
- RFC 9112 §3.2 [S, from RFC text]: server MUST answer 400 to a request with no Host, more than one Host line, or an invalid Host value. RFC 9110 §7.4: 421 for misdirected requests; host is case-insensitive, default port omitted (§4.2.3). https://www.rfc-editor.org/rfc/rfc9110.html , https://www.rfc-editor.org/rfc/rfc9112.html

### 3.2 Recommended Symfony config [I]
- FrankenPHP serves PHP directly → no proxy hop: `trusted_proxies` empty, `trusted_headers` empty. If a CDN/LB is added later, trust only its ranges and only `x-forwarded-for` and `x-forwarded-proto`; still not `x-forwarded-host` (Cloudflare passes the original Host anyway).
- `trusted_hosts` cannot list dynamic custom domains. Use: `['^([a-z0-9-]+\.)?colsynq\.app$', '^[a-z0-9.-]+$']`-style broad pattern (syntax guard only) and rely on (a) Caddy refusing TLS for unknown names (no cert), (b) `strict_sni_host on`, (c) the resolver's host classifier returning 404/421 for unknown hosts. Plain HTTP port 80 only redirects to HTTPS and serves ACME challenges.
- Absolute URLs: never `$request->getSchemeAndHttpHost()` for emails; use `TenantDirectory::canonicalUrl($tenantId)`; set `framework.router.default_uri: 'https://app.colsynq.app'` for console/workers (Symfony 8.1 deprecates `router.request_context.host`). https://symfony.com/doc/current/routing.html

### 3.3 Host normalisation and classification [I]

Normalise:
1. Take `Request::getHost()` (Symfony already strips port and lowercases; it validates characters).
2. Strip one trailing dot (`acme.colsynq.app.` is valid DNS and otherwise bypasses exact-match lookups/caches).
3. IDN: convert to ASCII with `idn_to_ascii($h, IDNA_NONTRANSITIONAL_TO_ASCII | IDNA_CHECK_BIDI | IDNA_USE_STD3_RULES, INTL_IDNA_VARIANT_UTS46)`; reject on false. Store and look up only the ASCII (punycode) form; store `host_unicode` for display. https://www.php.net/manual/en/function.idn-to-ascii.php
4. Reject length > 253, any label > 63, empty labels.

Classify (in this order):
| Class | Rule | X-Tenant |
|---|---|---|
| Platform host | exact list in config: `app.colsynq.app`, `api.colsynq.app`, `auth.colsynq.app`, `console.colsynq.app`, `colsynq.app`, `www.colsynq.app` | `api`/`app`: required for Required routes; `auth`/`console`: ignored or Platform |
| Tenant subdomain | `^([a-z0-9][a-z0-9-]{1,62})\.colsynq\.app$` and not reserved | optional; must equal host's tenant |
| Deeper subdomain | anything else under `colsynq.app` | → 404 (421 if over TLS without a cert) |
| Custom domain | lookup `tenant_domain.host` where status active/failing | optional; must match |
| Unknown | none of the above | 404 generic page, no tenant branding |

### 3.4 Reserved slugs, squatting, homoglyphs [I]
- Slug regex is ASCII only, so classic IDN homoglyphs are already impossible. Remaining risks: look-alikes (`rn` vs `m`, `0` vs `o`, `l` vs `1`, hyphen tricks), impersonating real cooperatives, and taking names of known societies.
- Reserved list (code catalogue, versioned): infrastructure labels (`www, api, app, auth, login, sso, admin, console, platform, static, cdn, assets, media, files, mail, smtp, imap, mx, ns1, ns2, status, docs, help, support, billing, pay, payments, webhooks, dev, staging, test, sandbox, internal, _acme-challenge, autodiscover, autoconfig, mta-sts`), brand words (`colsynq*`), regulator/bank names (`cbn, ncs, nibss, ndic, sec, efcc, firs` …), and anything starting `xn--`.
- Add `tenant.slug_skeleton` (lowercase, map `0→o, 1→l, rn→m, vv→w`, remove hyphens) with a unique index among non-archived tenants; collision → manual review. UTS #39 skeleton is the formal model if Unicode is ever allowed. https://www.unicode.org/reports/tr39/
- Society slugs are created by platform staff (5.1, `tenants:create`), so squatting risk is low today; keep the review step if self-service sign-up is added.

### 3.5 Public Suffix List [S + I]
- MDN [S]: site = registrable domain (PSL entry + one label); SameSite uses schemeful same-site. https://developer.mozilla.org/en-US/docs/Glossary/Site
- PSL guidelines [S]: private section is for owners issuing subdomains to mutually untrusting parties; needs `_psl` TXT kept forever; projects of a few thousand users likely declined; don't submit to dodge rate limits; propagation is slow and removal hard. https://github.com/publicsuffix/list/wiki/Guidelines
- Effect if `colsynq.app` were listed [I]: each `<slug>.colsynq.app` becomes its own site → SameSite and Sec-Fetch-Site would isolate tenants; `Domain=colsynq.app` cookies rejected; LE per-registered-domain limit becomes per tenant. But `app.`, `api.`, `auth.colsynq.app` would also become cross-site to each other, breaking shared cookies/CSRF assumptions between them.
- Recommendation [I]: do not apply now (too small; side effects). Design as if all subdomains are one site (section 4). Revisit if tenant count is large; alternative is to put tenant hosts on a separate registrable domain from platform/auth hosts, e.g. tenants on `colsynq.app`, auth on `colsynq.com` — only if the threat model needs it.

---

## 4. Cookies, CSRF, CORS, OAuth, CSP

### 4.1 Cookies [S]
- `__Host-`: Secure, no Domain, Path=/ — cannot be set or overwritten from another subdomain. `__Host-Http-` additionally requires HttpOnly. https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie
- OWASP Session: do not set Domain; use `__Host-`; SameSite Strict preferred or Lax; SameSite is defence in depth, not a replacement for CSRF tokens. https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- OWASP CSRF: "SameSite is scoped to the registrable domain, not the origin"; naive double-submit "remains vulnerable to cookie injection attacks, especially when attackers control subdomains"; Sec-Fetch-Site is the primary signal; treat `same-site` as allowed only if you trust sibling subdomains; fallback Origin check mandatory. https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- PortSwigger: a request can be same-site while cross-origin; a flaw on one sibling domain can break site-based defences for all. https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions

Recommended [I]:
- Every cookie host-only and `__Host-` prefixed (session, CSRF, oauth_tx, handoff). Consider `__Host-Http-` for the BFF session.
- Each host has its own BFF session; no cross-subdomain session sharing. Switching society in the browser = navigate to the other host + handoff (same as custom domains), or stay on `app.colsynq.app` and use X-Tenant.
- CSRF rule for unsafe methods: allow if `Sec-Fetch-Site: same-origin`; reject `same-site` and `cross-site`; if header absent, require `Origin` exactly equal to the request's own origin; plus synchronizer/signed CSRF token bound to the session.
- Tenant-controlled content (logo, brand files, uploads, any SVG/HTML) served from a separate registrable domain (e.g. `colsynq-files.app`, or object storage domain) with `Content-Disposition` and `X-Content-Type-Options: nosniff`, never from `*.colsynq.app`. `brand_color` validated as `#RRGGBB`.

### 4.2 CORS [S] https://portswigger.net/web-security/cors
- Misconfigurations: reflecting Origin with credentials; suffix/prefix/regex mistakes; trusting `null`; trusting HTTP origins or XSS-prone subdomains.
- Recommended [I]: SPA and BFF same-origin on each host → browser needs no CORS for authenticated calls. Where CORS is needed (public branding endpoint, mobile web views), allow-list = exact origins built from the resolved tenant: `https://<slug>.colsynq.app` + active `tenant_domain` hosts; compare parsed origin strings exactly; never `null`; always `Vary: Origin`; include `X-Tenant` in `Access-Control-Allow-Headers` only on shared hosts.

### 4.3 OAuth / OIDC / passkeys
- RFC 9700 [S]: exact string matching of redirect URIs (only localhost port exception for native apps); no open redirectors; mix-up defence via `iss` (RFC 9207). https://www.rfc-editor.org/rfc/rfc9700.html
- WebAuthn Related Origin Requests [S]: max 5 unique labels in practice → one RP ID cannot cover hundreds of custom domains; recommends a single shared RP ID and federation first. https://passkeys.dev/docs/advanced/related-origins/
- Recommended [I]:
  - All credential ceremonies (password, TOTP, passkey with RP ID `colsynq.app`, Google/Microsoft OIDC) happen on `auth.colsynq.app`. Only one redirect URI registered per IdP: `https://auth.colsynq.app/auth/oauth/callback`.
  - Flow from a custom domain: `https://portal.mysociety.ng/login` → BFF creates `return_host` + `code_verifier`, stores verifier in its `__Host-` cookie, redirects to `auth.colsynq.app/authorize?return_host=portal.mysociety.ng&code_challenge=…`. Auth server validates `return_host` against active `tenant_domain` / tenant subdomains (no open redirect), authenticates, issues handoff code (60 s, single use, bound to `user_id, sid, tenant_id, return_host, code_challenge`), redirects with 303 to `https://portal.mysociety.ng/auth/handoff?code=…`. Target BFF redeems back-channel with the verifier, gets tokens, sets its own `__Host-` session.
  - New table `auth_handoff` (code_hash PK, session_id, tenant_id, target_host, code_challenge, expires_at, used_at) or Redis with TTL; reuse → revoke session.

### 4.4 CSP [I]
- `default-src 'self'; script-src 'self' 'nonce-…'; frame-ancestors 'none'; form-action 'self' https://auth.colsynq.app; img-src 'self' https://<files-domain>; connect-src 'self'`; `base-uri 'none'`. Same policy on custom domains (with `'self'` resolving to the custom origin).

---

## 5. API clients, X-Tenant, rate limiting, caching

- Precedence [I]:
  1. Classify host (3.3). Unknown → 404.
  2. Tenant-bound host (subdomain or custom domain): tenant = host. X-Tenant optional; if present and different → 400 `tenant_mismatch`.
  3. Shared host (`api.`, `app.`): tenant = X-Tenant; missing on a Required route → 400 `tenant_required`.
  4. X-Tenant value is a slug; also accept the tenant id (UUID) so mobile clients survive slug renames. Normalise the same way.
  5. Never from JWT, body, or query (as v3.1).
- Mobile: store tenant id, send `X-Tenant: <id>`; tokens are tenant-free so switching is a header change (keeps 9.2).
- API keys: tenant comes from the key's profile; if X-Tenant/host disagree with the key's tenant → 403 (not 400, to avoid probing).
- Rate limiting [I]: two layers. (a) Before resolution: per IP for unknown host/slug misses (stops scans), per IP for auth endpoints. (b) After resolution: per tenant (`t:<tenant_id>:rl:*`) and per principal. Symfony RateLimiter keys follow 8.6 prefix rule.
- Caching [S RFC 9110 §12.5.5: Vary expands the cache key]: shared caches key on Host by default; add `Vary: X-Tenant, Origin` (and `Authorization` is not cacheable by shared caches by default). Recommended: tenant data `Cache-Control: private, no-store`; only public branding/static assets are cacheable. API Platform HTTP cache invalidation tags must be tenant-prefixed (already noted in db_per_tenant_symfony.md).

---

## 6. Resolution flow, caching, renames, suspension

### 6.1 Flow [I]
```
kernel.request
 256 ValidateRequestListener (Symfony: checks trusted hosts / conflicting forwarded headers)
  32 RouterListener → _tenant requirement known (404 if no route)
  20 TenantResolveListener:
       host = normalise(getHost())
       class = classify(host)              # platform | subdomain | custom | unknown
       hint  = X-Tenant (normalised)
       rule table (section 5) → slug|id|host key
       ResolvedTenant = TenantDirectory (cache → landlord)
       status checks per requirement → 404 / 423 / 400
       TenantContext::set()
   8 Firewall
```
- Note: unknown host + unknown route returns the router 404 first; fine, both are 404 and generic.
- Error responses before/at resolution must not render tenant branding unless resolution succeeded.

### 6.2 Cache [I]
- Keys: `tenant:v1:slug:<slug>`, `tenant:v1:id:<id>`, `tenant:v1:host:<ascii-host>`.
- APCu is per process/node; "delete at once" cannot reach other nodes. Use Redis as the shared layer (TTL 60 s) and either no APCu, or APCu with 5 s TTL, or a Redis pub/sub "tenant changed" message that clears APCu on every node. Symfony `TagAwareCacheInterface` with tag `tenant_<id>` purges all keys of a tenant at once.
- Negative cache: only after classification says "syntactically valid tenant subdomain / custom host"; TTL 30–60 s; Redis LRU bounded. On domain activation or slug creation, delete the negative key explicitly (otherwise a just-verified domain returns 404 for up to a minute).
- Cache stampede: single-flight (lock) on miss.

### 6.3 Mapping to the DB connection
- Unchanged from v3.1 section 8: ResolvedTenant holds id only; `TenantConnectionResolver` maps id → `tenant_database`. Host/slug never reach the DB layer. Good.

### 6.4 Slug rename [I]
- `tenant_slug_history(slug PK, tenant_id, retired_at, redirect_until, reserved_until)`.
- Old slug: 308 redirect from `<old>.colsynq.app` to `<new>.colsynq.app` (same path) for 90 days; X-Tenant with old slug accepted with a `Deprecation`/`Link` header for 90 days.
- Old slug never assigned to another society (or quarantined ≥ 12 months) to prevent link hijack from old emails/QR codes.
- Rename invalidates caches (`tag tenant_<id>`), triggers re-issue of nothing (wildcard covers it).

### 6.5 Suspended / archived [I]
- Suspended: resolver still resolves (so the user sees a branded "suspended" page and admins can export or contact support); Required routes → 423 problem+json `type: https://colsynq.app/problems/tenant-suspended` (or 403 — 423 is WebDAV "Locked", acceptable but unusual). TLS keeps working (ask endpoint allows).
- Archived: custom domains `inactive`; subdomain returns 404 or a generic "no longer available" page; read-only export only via platform.
- Provisioning/failed: 404 to the public.

### 6.6 Email and links [I]
- `canonical_host` derived: primary active custom domain else `<slug>.colsynq.app`. Links in emails and push notifications built from it in workers via `TenantDirectory`, never from the request. Signed URLs should include the host in the signature so a link for tenant A cannot be replayed on tenant B's host.
- Auth emails (password reset, email verification) point at `auth.colsynq.app` with a `return_host` the auth service validates.

---

## 7. Failure modes and fixes

| Failure | Cause | Fix |
|---|---|---|
| Wildcard subdomain issuance hits LE 50/week | On-demand enabled for `*.colsynq.app` names | Wildcard cert via DNS-01; on-demand only for custom domains; ask endpoint rejects anything under `colsynq.app` |
| Random hostnames trigger issuance (cost, rate limits, LE pause) | Unrestricted on-demand or ask that just says yes | Ask reads verified `tenant_domain` only; internal endpoint; per-name rate limit |
| Customer leaves, cert renewal keeps failing, LE pauses identifier | No re-check | Daily DNS re-check; `inactive` stops renewal |
| Domain fronting: SNI = tenant A, Host = tenant B | Default Caddy | `strict_sni_host on` → 421 |
| `acme.colsynq.app.` or `ACME.colsynq.app` bypasses cache/lookup | Missing normalisation | Normalise (3.3) |
| X-Forwarded-Host poisoning reset links | Trusted override header / Host-based URLs | Do not trust it; links from directory |
| Just-verified domain still 404 | Negative cache | Purge negative key on activation |
| Stale tenant after suspend on other nodes | APCu per node | Shared Redis or pub/sub invalidation |
| Old slug taken by another society | Slug reuse | `tenant_slug_history`, never reuse |
| Tenant A subdomain CSRFs tenant B | Same-site siblings | Sec-Fetch-Site same-origin rule + CSRF token + `__Host-` cookies |
| Uploaded SVG XSS on `<slug>.colsynq.app` | User content same site | Separate files domain |
| Enumeration of societies via 404/423/403 differences | Different codes | Accept for subdomains (slugs are public names), but uniform 404 for provisioning/failed/archived; rate-limit misses per IP; 403 without tenant hint after auth (v3.1 7.4 already) |
| Unknown slug under wildcard DNS returns a page that looks like a valid tenant | Generic SPA served for any host | Server-side check before serving SPA shell; unknown → 404 page with no branding, `noindex` |
| Passkey fails on custom domain | WebAuthn RP ID mismatch | Ceremonies on `auth.colsynq.app` + handoff |
| Customer HSTS `includeSubDomains` set by us breaks their intranet | Wrong HSTS on custom domains | No includeSubDomains on custom hosts |
| CAA on customer apex blocks LE | Customer CAA | Verifier checks CAA and shows the error before activation |
| Resolver runs before router and lacks requirement | Priority | Keep 20 (after router 32, before firewall 8); test with `debug:event-dispatcher kernel.request` in CI |
| `trusted_hosts` left empty | Default accepts any Host | Syntax-guard pattern + classifier + strict SNI |

---

## 8. Nigeria-specific notes [I]
- Many societies will use `.com.ng` / `.org.ng` / `.ng` (NiRA). `com.ng`, `org.ng` are public suffixes → validation must reject the bare suffix and use the PSL to find the registrable domain (library such as `jeremykendall/php-domain-parser` — dependency change needs approval).
- Some Nigerian registrars' DNS panels lack ALIAS/ANAME and underscore labels sometimes confuse users → offer the HTTP verification fallback and a "check now" button; recommend a subdomain (`portal.`) over apex.
- Latency: consider a CDN with Lagos PoP later (Cloudflare).

---

## 9. Proposed table changes

### tenancy.tenant_domain (replace)
| Column | Notes |
|---|---|
| id uuid PK | v7 |
| tenant_id uuid | |
| host text | ASCII/punycode, lowercase, no trailing dot; unique where `removed_at is null` |
| host_unicode text | display |
| status text | `pending, active, failing, inactive, removed` (CHECK) |
| status_reason, status_changed_at | naming rule |
| is_primary bool | partial unique (tenant_id) where is_primary and status = active |
| redirect_to_primary bool | www/apex pair |
| verification_method text | `dns_txt, http` |
| verification_token text | plain; public by design (lives in DNS); 128-bit base32 |
| verified_at, last_checked_at, last_check_ok_at timestamptz | |
| check_failures int | |
| routing_type text | `cname, a_record` |
| cert_provider text | `caddy, cloudflare, cloudfront` |
| cert_status text | `none, pending, issued, failed` |
| cert_expires_at timestamptz, cert_last_error text | |
| external_ref text | e.g. Cloudflare custom hostname id |
| created_by_profile_id uuid, created_at, updated_at, removed_at | |

### tenancy.tenant_slug_history (new)
`slug PK, tenant_id, retired_at, redirect_until, reserved_until`

### tenancy.tenant (add)
`slug_skeleton text` (unique among non-archived) — optional.

### iam.auth_handoff (new, or Redis)
`code_hash PK, session_id, user_id, tenant_id, target_host, code_challenge, expires_at, used_at`

Events: `TenantDomainActivatedV1`, `TenantDomainDeactivatedV1`, `TenantSlugChangedV1` (cache purge, IAM copy of allowed return hosts if IAM must validate `return_host` without reading tenancy tables — keeps the "IAM never reads tenancy tables" rule).

---

## 10. Sources
1. https://letsencrypt.org/docs/rate-limits/
2. https://isrg.org/post/from-90-to-45/
3. https://letsencrypt.org/docs/integration-guide/
4. https://www.rfc-editor.org/rfc/rfc9773.html
5. https://letsencrypt.org/2025/06/26/expiration-notification-service-has-ended/
6. https://letsencrypt.org/docs/caa/ ; https://www.rfc-editor.org/rfc/rfc8657.html
7. https://caddyserver.com/docs/automatic-https ; https://caddyserver.com/docs/caddyfile/options
8. https://frankenphp.dev/docs/config/
9. https://datatracker.ietf.org/doc/draft-ietf-dnsop-domain-verification-techniques/
10. https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/domain-support/hostname-validation/ (+ pre-validation, realtime-validation, remove-custom-hostnames, plans, apex-proxying pages)
11. https://aws.amazon.com/blogs/aws/reduce-your-operational-overhead-today-with-amazon-cloudfront-saas-manager
12. https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html
13. https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages
14. https://vercel.com/docs/domains/working-with-domains/add-a-domain
15. https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/Subdomain_takeover
16. https://hstspreload.org/ ; https://www.thesslstore.com/blog/google-launches-dot-app/
17. https://symfony.com/doc/current/deployment/proxies.html ; https://symfony.com/doc/current/reference/configuration/framework.html ; https://symfony.com/doc/current/routing.html
18. https://portswigger.net/web-security/host-header
19. https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
20. https://www.rfc-editor.org/rfc/rfc9110.html ; https://www.rfc-editor.org/rfc/rfc9112.html
21. https://www.php.net/manual/en/function.idn-to-ascii.php ; https://www.unicode.org/reports/tr39/
22. https://github.com/publicsuffix/list/wiki/Guidelines ; https://developer.mozilla.org/en-US/docs/Glossary/Site
23. https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie ; https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-Fetch-Site
24. https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html ; https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
25. https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions ; https://portswigger.net/web-security/cors
26. https://www.rfc-editor.org/rfc/rfc9700.html
27. https://passkeys.dev/docs/advanced/related-origins/
