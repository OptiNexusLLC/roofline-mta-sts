# roofline-mta-sts

MTA-STS policy publication for **Hayden Building Maintenance Corp. — roofline.com**, maintained in [OptiNexusLLC/roofline-mta-sts](https://github.com/OptiNexusLLC/roofline-mta-sts).

**Release rule: publish and verify the HTTPS policy first; update the DNS TXT policy identifier second.** Keep mail delivery compatible with policies that senders may still have cached.

## 1. Scope and recorded baseline

Repository configuration recorded on **November 8, 2026**:

- Default branch: `main`.
- Latest recorded commit date: October 24, 2026.
- Policy file: `.well-known/mta-sts.txt`.
- Custom-domain file: `CNAME`, containing `mta-sts.roofline.com`.
- Static-publication marker: `.nojekyll`.

Recorded policy contents:

```text
version: STSv1
mode: testing
mx: roofline-com.mail.protection.outlook.com
max_age: 604800
```

This is a **repository baseline, not a production health check**. The live DNS records, Pages publishing settings, public policy response, HTTPS certificate, and receiving SMTP TLS configuration must be checked separately before each release.

The policy describes inbound mail transport expectations for supporting sending mail servers. It does not configure Microsoft 365 mail routing or replace SPF, DKIM, or DMARC.

## 2. Files and external controls

| Item | Purpose |
| --- | --- |
| `.well-known/mta-sts.txt` | Policy body intended for the public HTTPS endpoint. |
| `.nojekyll` | Disables Jekyll processing for branch-based static publication. Preserve it and verify that the dot-prefixed `.well-known` directory reaches the published site. |
| `CNAME` | Custom-domain file for branch-based Pages publication. It is not a DNS record. Custom Actions publication uses Pages settings and ignores this file. |
| `README.md` | This operating procedure; it is not the policy. |
| GitHub **Settings → Pages** and deployment workflow | Select the actual publishing source and deploy the website. The default branch alone does not establish that source. |
| DNS for `mta-sts.roofline.com` | Resolves the HTTPS policy host. Managed outside these policy files. |
| DNS TXT at `_mta-sts.roofline.com` | Advertises MTA-STS and signals policy updates through its `id`. |
| MX records for `roofline.com` and receiving SMTP servers | Determine the actual inbound delivery destinations and their TLS behavior. |

A repository commit does **not** automatically update external DNS. Do not assume DNS automation exists.

Preserve the established publishing setup during routine policy releases. Any hosting, custom-domain, DNS routing, or workflow change requires its own reviewed plan. For a custom Pages workflow, inspect the deployed artifact: it must include `.well-known/mta-sts.txt` at the site root.

## 3. Prepare a pull request

1. Create a branch from current `main`; submit a pull request rather than editing the release branch directly.
2. Describe the reason, exact policy diff, expected delivery effect, and affected MX hosts. Keep unrelated changes out.
3. Record the current public policy, DNS TXT value and TTL, MX answers and TTLs, deployment commit, and any older policy cache windows still relevant. Save the last known-good policy.
4. Name a release operator, reviewer, DNS administrator, and mail administrator. Record the rollout window, stop conditions, monitoring plan, and recovery steps.
5. Reserve a fresh DNS policy identifier for this release. Do not publish it yet.
6. Obtain reviewer approval. Obtain mail-administrator approval for changes to `mx`, `mode`, or `max_age`; coordinate the DNS step with the DNS administrator.

These are operating requirements for this repository, not a claim that GitHub branch protection or approval rules are already configured.

### Repository validation

Review the complete policy, not only the changed lines:

- Use plain UTF-8 text without a BOM, one `key: value` per line, and LF or CRLF line endings. Keep `version: STSv1` first as a repository convention.
- Require exactly one `version`, `mode`, and `max_age`. Permit repeated `mx` lines.
- Use only `testing`, `enforce`, or `none` for `mode`. Require at least one `mx` for `testing` and `enforce`.
- Use a decimal integer from `0` through `31557600` for `max_age`. Retain `604800` unless a separate cache-lifetime change is justified and approved.
- Use reviewed MX hostnames, without URL schemes, paths, ports, priorities, or IP literals. If a wildcard is necessary, it may replace only the entire leftmost label, such as `*.example.net`. Do not broaden matching merely to silence failures.
- Exclude comments, Markdown fences, duplicate singleton fields, and unexpected fields as a repository hygiene rule.
- Preserve `.nojekyll`, `CNAME`, and the policy path.

Run any configured syntax checks and review their output. If none exist, document a manual syntax review; do not claim an automated check ran.

**Passing GitHub Actions checks validates only what those checks actually test. It does not establish production DNS correctness, Pages availability, certificate validity, or SMTP readiness.**

## 4. Verify receiving MX hosts before changing matching

Have the mail administrator verify:

1. The authoritative MX answers for `roofline.com`, including backup or lower-priority destinations, agree with the intended Microsoft 365 configuration and any approved mail gateway.
2. Every intended destination matches the proposed policy.
3. Every destination supports SMTP STARTTLS and presents an unexpired certificate with a trusted chain and a SAN matching its **MX hostname**. An HTTPS certificate for the policy host does not satisfy this SMTP requirement.
4. External SMTP/TLS checks cover each destination and its advertised addresses, including IPv6 where published. Record hostname verification results; a successful TCP connection or unverified TLS handshake is insufficient.
5. Representative external test messages reach monitored recipients. Record message traces and investigate delays or non-delivery reports.

The recorded `roofline-com.mail.protection.outlook.com` policy entry is not evidence that the current production MX answer still has that value.

### Plan overlap for an MX migration

Do not replace an old allowed host and move production mail routing in one step.

1. While the old mail path remains operational, publish a transitional policy allowing both the old and approved new destinations. Verify HTTPS, then update the TXT `id` using Section 6.
2. Allow restrictive older policies to age out before routing mail to a destination they would reject. Use the cache assessment in Section 7.
3. Change MX routing under a separately approved mail-migration plan. Account for the old MX DNS TTL and queued delivery retries; keep the old path available for the documented transition.
4. After old routing is no longer needed, remove obsolete policy entries through another reviewed policy release.

An approved, tested transition plan is required even if the repository currently says `testing`: a previously served `enforce` policy may still exist in sender caches.

## 5. Publish and verify HTTPS — first release gate

After approvals:

1. Merge the policy PR.
2. Confirm the Pages deployment for the intended commit succeeds. Check the configured source in **Settings → Pages**; do not assume that merging into `main` publishes it.
3. Fetch the exact public URL from outside the hosting environment:

   [https://mta-sts.roofline.com/.well-known/mta-sts.txt](https://mta-sts.roofline.com/.well-known/mta-sts.txt)

4. Require all of the following before authorizing the DNS step:

   - Direct **HTTP 200** from that exact HTTPS URL; no redirects.
   - `Content-Type: text/plain` with an appropriate charset, if specified.
   - A trusted, unexpired HTTPS certificate valid for `mta-sts.roofline.com`.
   - The complete intended policy body, matching the approved release; no HTML error page or authentication requirement.
   - Correct public host resolution and working HTTPS over all published address families.

Example inspection command for a shell with `curl`:

```sh
curl --proto '=https' --tlsv1.2 --fail --silent --show-error \
  --max-time 60 \
  --dump-header policy.headers \
  --output policy.live.txt \
  --write-out 'HTTP %{http_code}\n' \
  https://mta-sts.roofline.com/.well-known/mta-sts.txt
```

Inspect the headers and body and compare the body with the released file, allowing only LF/CRLF differences. This command writes local evidence files. It does not change production.

Do not add `-L` to follow redirects or `-k` to bypass certificate checks. A successful curl exit alone is insufficient: explicitly check for HTTP 200 and the expected body.

Repeat the check from another external network or independent monitoring service. If results are stale, inconsistent, or incorrect, **stop before changing DNS**. A GitHub file view or raw-content URL is not a substitute for the production URL.

## 6. Update the DNS TXT identifier — second release gate

Only after Section 5 passes, have the authorized DNS administrator update the effective TXT record at:

```text
_mta-sts.roofline.com
```

Illustrative value only — **not a statement of the current DNS record**:

```text
v=STSv1; id=20261108T150000Z;
```

Use a fresh identifier for every policy revision, including recovery releases. The identifier is 1–32 ASCII letters or digits; it is an opaque change token, not a policy hash or an ordered version number. Do not put it inside `mta-sts.txt`.

Update the existing effective MTA-STS record rather than adding a second competing `v=STSv1` record. If discovery is delegated through a CNAME, coordinate with the owner of the effective TXT record. DNS-provider interfaces may expect `_mta-sts` rather than the fully qualified name; verify the resulting name.

RFC 8461 Section 8.1 recommends **HTTPS first, TXT second**. Reversing the order can cause a sender to associate the new identifier with an old HTTPS body.

Record the DNS change time and verify authoritative answers and independent recursive resolvers. Example read-only queries:

```sh
dig +noall +answer roofline.com NS
dig +noall +answer roofline.com MX
dig +noall +answer _mta-sts.roofline.com TXT
dig +noall +answer mta-sts.roofline.com CNAME
dig +noall +answer mta-sts.roofline.com A
dig +noall +answer mta-sts.roofline.com AAAA
```

Query each authoritative nameserver directly as well as recursive resolvers; retain TTLs and follow any delegation. An absent CNAME alone is not a failure if the approved hosting setup uses another valid record type.

Repeat the HTTPS checks after the DNS update. Monitor inbound delivery and TLS reports through the agreed observation window.

README-only changes do not require a policy TXT identifier update unless the served policy body also changes.

## 7. Caching, timing, and rollback limits

Treat these as separate clocks:

| Clock | Operational consequence |
| --- | --- |
| DNS TXT TTL | Resolvers can continue returning the old policy identifier. Lowering the TTL does not shorten answers already cached under the previous TTL. |
| Policy `max_age` | The recorded `604800` is seven days from a sender's successful policy fetch, not from the Git commit. Different senders have different expiration times. |
| MX and policy-host DNS TTLs | Old routing or hosting answers can persist independently of TXT discovery. |
| Deployment and edge propagation | A successful build does not establish that all public responses contain the new policy. |

A new TXT identifier signals an update; it does not flush every sender's policy cache. A sender unable to retrieve a replacement can continue using a valid cached policy.

Reducing `max_age` affects newly fetched policies. It does not shorten the lifetime of copies already cached with a longer value. For an incompatible change, first release the shorter lifetime while preserving compatibility, then allow older copies to expire before depending on it.

For each previously served policy, assess **last possible fetch time + that policy's max_age**. Include older revisions with longer lifetimes and any period during which stale HTTPS copies could still be served. If the history is uncertain, do not assume seven days is sufficient; resolve the uncertainty before an incompatible change.

Use those deadlines, relevant DNS TTLs, and observed delivery behavior to set the overlap period. Neither waiting one TXT TTL nor reverting a commit guarantees immediate recovery.

## 8. Gate for changing testing to enforce

Treat enforcement as a separate, explicitly approved release. Do not combine it with an MX migration or unrelated hosting changes.

Before approval, require:

- All repository, HTTPS, DNS, MX, and SMTP checks above are recorded and passing.
- A separately configured and verified TLS reporting destination at `_smtp._tls.roofline.com`, with an owner actively reviewing received reports. Do not assume testing mode creates reporting automatically.
- Representative inbound traffic observed in testing for an agreed period covering ordinary business traffic and multiple reporting cycles. For this repository, plan at least seven days after verified publication and DNS visibility; extend for low volume or unresolved findings. This is an operating minimum, not an RFC guarantee.
- No unexplained policy-fetch, MX-matching, STARTTLS, or certificate failures. Missing reports do not prove success; consider sender participation and supplement with active checks.
- An approved cache-aware recovery plan and available operators.
- Explicit approval from the mail administrator and the business owner responsible for email availability.

In `testing`, policy failures do not themselves require senders to withhold delivery. In `enforce`, supporting senders must withhold delivery to noncompliant destinations; prolonged failure can lead to delays and eventual non-delivery.

Release `mode: enforce` through the same **HTTPS verification → fresh TXT id → monitoring** sequence. Do not raise `max_age` in that initial enforcement release.

## 9. Recover from an incorrect publication

1. Stop additional rollout steps and alert the mail administrator. Capture the served policy, effective TXT identifier, timestamps, deployment commit, and delivery symptoms.
2. If the TXT identifier has not changed, stop that step. Restore or correct HTTPS promptly: some senders may already have fetched the bad policy without a new identifier.
3. Prefer restoring a working SMTP/TLS path compatible with cached policies. A corrected website cannot immediately repair every sender's cached restrictions.
4. Prepare a reviewed recovery PR using the last known-good policy **only if it remains compatible with current mail routing**; otherwise prepare a compatible corrective policy.
5. Deploy and verify the recovery policy at the public URL. Then publish a **new, unused TXT identifier**, even if restoring previously used policy contents.
6. Continue external checks, mail-flow monitoring, and TLS-report review through the assessed cache and DNS windows. Record when compatibility and delivery are restored.

Switching back to `testing` may help senders that fetch the recovery, but does not instantly replace cached `enforce` policies.

Do not delete the TXT record, HTTPS endpoint, `CNAME`, or repository as an emergency bypass. For an intentional retirement, use the staged `mode: none` procedure in RFC 8461 Section 8.3 and keep discovery and HTTPS available until previously served policies have expired.

## 10. Required release evidence

Keep a release record in the PR and the approved internal change log. Redact recipient details and sensitive diagnostics from public records.

```text
Change / PR:
Reason and intended delivery effect:
Release operator / reviewer:
Mail administrator / DNS administrator:
Business approval, if enforcing:
Previous and new policy bodies / max_age:
Relevant older policies and expiration assessment:
Previous and new effective TXT id / TTL:
Authoritative and recursive DNS results / check times:
MX inventory and external SMTP/TLS results:
Pages source / deployment URL / deployed commit:
HTTPS verification times, status, headers, certificate and body comparison:
External mail-flow tests / message-trace references:
TLS reporting configuration / report-review findings:
HTTPS verification gate passed at:
TXT update performed by / time:
Observation window / stop conditions:
Recovery policy and cache-compatible mail path:
Final verification / remaining risks / incident follow-up:
```

Use timestamps with an explicit timezone, preferably UTC. Do not mark a release complete from CI results alone.

## References

- [RFC 8461 — MTA-STS](https://www.rfc-editor.org/rfc/rfc8461.html), especially Sections 3–5, [8.1: Policy Updates](https://www.rfc-editor.org/rfc/rfc8461.html#section-8.1), and [8.3: Removing MTA-STS](https://www.rfc-editor.org/rfc/rfc8461.html#section-8.3).
- [RFC 8460 — SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460.html).
- [Microsoft: Enhance mail flow with MTA-STS](https://learn.microsoft.com/en-us/exchange/security-and-compliance/enhance-mail-flow-using-strict-transport-security).
- [GitHub: Creating a Pages site — static site generators](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site#static-site-generators).
- [GitHub: Managing a custom domain](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site).
- [GitHub: Securing a Pages site with HTTPS](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https).
