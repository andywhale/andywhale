---
title: "Security-Focused Drupal 11: A Few Essential Modules and Configuration for Production Sites"
description: "Drupal 11 is secure by default, but production sites need more. Here are some essential security modules and configurations that should be on every site's checklist before launch."
date: 2026-04-18T13:21:43+01:00
image: "images/drupal-security-padlock.svg"
draft: false
type: "post"
tags: ["drupal", "drupal11", "security", "modules", "authentication", "compliance"]
---

Drupal's core security is excellent. The team takes vulnerabilities seriously, releases patches promptly, and the framework is designed with security principles in mind. But out-of-the-box Drupal assumes a baseline of security that production sites - especially those handling user data, authentication, or sensitive content - need to exceed.

This article covers the security modules and configurations that should be on every Drupal 11 site's checklist. These aren't afterthoughts. They're essential layering on top of Drupal's foundation.

## What Drupal 11 Brings to the Table

Before discussing what you need to add, it's worth understanding what Drupal 11 provides out of the box.

Drupal 11 ships with a significantly hardened foundation compared to earlier versions. The platform now runs on PHP 8.3 and Symfony 7, both modern frameworks with robust security practices and regular patching. This alone eliminates classes of vulnerabilities that plagued older stacks.

The core includes improved dependency management through Composer, with better vetting of third-party packages. Drupal 11 also introduces stricter media and file permissions by default - files aren't globally readable unless explicitly required, and permission structures around uploads are much clearer.

For access control, Drupal 11 brings contextual permission checking, the ability to disable the superuser account entirely (so you can operate with granular permissions instead), and improved audit capabilities for tracing who changed what.

Long-term support extends to December 2026 for Drupal 11.3.x, giving organisations a clear maintenance window and predictable patch cycles.

This is a genuinely secure foundation. But production sites handle real data, real users, and real risk. The modules and configurations below build on this foundation to address the specific threats that affect public-facing, user-authenticated sites.

## Authentication: SSO and MFA

The single biggest security win for most organisations is moving away from password-only authentication.

**SSO (Single Sign-On)** offloads authentication entirely to a trusted provider - your organisation's identity system, Google Workspace, Microsoft Entra ID, or similar. Users never create a Drupal password. The Drupal site becomes a relying party that trusts the external provider. This is transformative:

- Users authenticate once against a system your organisation already trusts and audits
- Drupal has no password to leak
- Credential rotation happens outside your site
- Compliance with organisational security policies is automatic

**MFA (Multi-Factor Authentication)** adds a second verification step. Even if a password is compromised, an attacker can't log in without the second factor. This is the most cost-effective security improvement for password-authenticated accounts.

Together, they're better. SSO removes the password problem entirely. MFA mitigates the risk when SSO isn't possible. For most Drupal sites, implementing at least one is non-negotiable. For sensitive sites (admin panels, membership systems, anything handling PII), implementing both is the standard.

Which modules? That depends on your infrastructure. If you're on Google Workspace, look at OAuth2 modules. If you're on Microsoft, Azure AD integration. For SAML-based systems (common in enterprises), SAML Service Provider modules are standard. There's no single "best" module because authentication is environment-specific. Evaluate based on what your organisation already trusts.

## HTTP Security Headers and XSS Protection: Security Enhancement Toolkit (SecKit)

SecKit is one of the most important security modules in the Drupal ecosystem, yet many sites don't use it.

What does it do? It adds HTTP security headers that tell browsers how to behave:

- **Content-Security-Policy (CSP)** - Restricts where scripts, stylesheets, and media can be loaded from. Prevents injected scripts from running even if XSS lands on your page.
- **X-Frame-Options** - Prevents your site from being embedded in iframes on other sites (clickjacking defence).
- **X-Content-Type-Options: nosniff** - Prevents MIME type sniffing, reducing attack surface.
- **Strict-Transport-Security (HSTS)** - Forces HTTPS connections and prevents protocol downgrade attacks.
- **Referrer-Policy** - Controls how much information is leaked when users click to external sites.

These aren't theoretical. Modern browsers understand and enforce these headers. A well-configured CSP, for example, can make stored XSS vulnerabilities entirely ineffective - even if malicious JavaScript lands in your database, it won't execute if it violates the policy.

**The trade-off:** CSP requires careful configuration. Too strict and your site breaks (external fonts stop loading, analytics breaks, custom scripts fail). Too loose and you lose the protection. Budget time for testing and iterative refinement.

SecKit lets you configure all of this through the UI. Start conservative, test thoroughly, then gradually tighten policies. Monitor your browser console for CSP violations to find what needs adjustment.

If you're not using SecKit, you're leaving XSS mitigation on the table.

## Username Enumeration Prevention

This is a subtle attack, but it matters: a poorly configured login form can leak whether a username exists on your site.

Here's the attack: An attacker submits a login form with a username they suspect. If they get "Password incorrect", they've confirmed the username exists. If they get "User not found", that username isn't registered. Over time, they build a list of valid usernames, which they can then target with brute force or sell.

Username Enumeration Prevention (drupal/username_enumeration_prevention) solves this by standardising error messages. Whether the username doesn't exist or the password is wrong, the user sees the same message: "Unrecognised username or password". An attacker can't tell the difference.

This also extends to registration forms, password reset forms, and other endpoints where username existence might be leaked.

It's a small module with a big impact on a specific, real attack vector. If your site has user accounts that matter (members, customers, subscribers, editors), this is essential.

## Password Complexity and Expiration: Password Policy

Password Policy lets you enforce password strength requirements and expiration policies. You can require:

- Minimum length
- Character variety (uppercase, lowercase, numbers, special characters)
- Expiration (passwords must be changed every N days)
- History (users can't reuse recent passwords)

**Why it matters:** If you're not using SSO or MFA, password strength is your primary defence. A weak password policy means even your authentication system is a liability.

**Why it's controversial:** Forced password expiration can paradoxically reduce security - users write passwords on sticky notes if they have to remember new ones every 30 days. The current NIST guidance recommends against periodic expiration unless there's evidence of compromise. Instead, enforce strong initial passwords and only force change when breached.

The module lets you configure this balance. Use it to set reasonable requirements: strong initial passwords, and expiration only in breach scenarios. Most security frameworks (SOC 2, PCI-DSS, NHS DSP Toolkit, ICO guidance) require password policies, so even if you personally disagree with forced rotation, compliance might require it.

## Spam and Form Security: Webform Spam Words

Drupal's webform system is powerful, but it's also a vector for spam. Attackers submit thousands of forms with spam content - selling products, advertising services, injecting links.

Webform Spam Words adds a configurable list of blocked words and patterns. When a submission contains a spam word, it's automatically rejected or quarantined. You can maintain your own list or use a curated one.

It's not perfect (spammers evolve their tactics), but combined with CAPTCHA or Cloudflare Turnstile, it dramatically reduces spam reaching your site. The module is lightweight and can save hours of manual spam moderation.

If you have public webforms, this is essential. If your forms are behind authentication or low-traffic, it might be less critical.

Cloudflare Turnstile is particularly effective - it uses challenge-based verification (invisible by default) rather than image recognition, and it doesn't require users to solve puzzles. The Drupal Webform Captcha or Turnstile modules integrate seamlessly.

## Session Management: Force Users Logout

Force Users Logout gives administrators the ability to terminate user sessions - logging out specific users or all users at once.

When does this matter?

- A user's account is compromised - log them out immediately instead of waiting for them to log out naturally
- A staff member leaves the organisation - immediately revoke their access instead of waiting for cookies to expire
- A security incident occurs - globally log out all users to force re-authentication

The module is simple but critical for incident response. Without it, you have to wait for sessions to naturally expire, during which a compromised account can still cause damage.

Combined with audit logging and monitoring, it transforms your ability to respond to security incidents.

## Audit and Compliance: Admin Audit Trail

Admin Audit Trail logs administrative actions - who changed what, when, and from where. You get a complete record of configuration changes, user modifications, permission updates, and more.

Why it matters:

- **Compliance:** Most security frameworks (SOC 2, HIPAA, PCI-DSS) require audit logs of administrative activity
- **Incident investigation:** If something goes wrong, you can trace exactly what changed and who made the change
- **Accountability:** Users know their actions are logged, which discourages misuse
- **Forensics:** If you're breached, audit logs help you understand what happened and what was exposed

The module integrates with Drupal's logging system. Configure it to send logs to a remote syslog server or aggregation tool - if your site is compromised, local logs might be deleted. External logs are your evidence.

Even if compliance isn't a requirement, audit logging is good practice. The overhead is minimal and the value in incident response is significant.

## Putting It Together: A Security Checklist

For a production Drupal 11 site, your security checklist should look like this:

**Authentication**
- [ ] SSO or MFA (or both) implemented
- [ ] Credentials stored securely (external system, not Drupal database)
- [ ] Session timeouts configured (shorter for sensitive data, longer for public sites)

**Headers and XSS**
- [ ] SecKit installed and configured
- [ ] CSP policy defined and tested
- [ ] HSTS enabled (if on HTTPS, which you should be)

**User Security**
- [ ] Username Enumeration Prevention installed
- [ ] Password Policy configured (or SSO removes this need)
- [ ] Account lockout after N failed login attempts

**Data Quality**
- [ ] Webform Spam Words configured (if accepting public input)
- [ ] Cloudflare Turnstile or CAPTCHA on public forms
- [ ] Input validation on all user-submitted content

**Incident Response**
- [ ] Force Users Logout module installed
- [ ] Admin Audit Trail logging all administrative changes
- [ ] Logs sent to remote storage
- [ ] Incident response plan documented

**Ongoing**
- [ ] Security update subscription enabled
- [ ] Regular security module updates applied
- [ ] Periodic audit log review
- [ ] Penetration testing (annually for sensitive sites)

## The Philosophy

Security in Drupal is layered. Core provides the foundation. These modules add essential protections. But they're not a substitute for good practices:

- Keep Drupal and modules updated
- Use strong hosting with regular backups
- Monitor for suspicious activity
- Have an incident response plan
- Educate your team

A well-configured, well-maintained Drupal site is genuinely secure. These modules are the difference between "default secure" and "production ready".

## Important Disclaimer: This Is Not a Security Audit

Before deploying any of these recommendations, understand what this article is and isn't:

**This article is:**
- A starting checklist of common security hardening measures for Drupal 11 sites
- A guide to modules that address specific threat vectors
- An introduction to security considerations that production sites should evaluate

**This article is NOT:**
- A comprehensive security guide for Drupal
- A substitute for a professional security audit
- Tailored to your specific threat model, infrastructure, or compliance requirements
- A guarantee that implementing these modules will secure your site

Security is context-dependent. What's essential for a membership site handling payment data is different from what's needed for a public marketing site. What's required for HIPAA compliance is different from PCI-DSS. What matters for a site with a small authenticated user base differs from a public-facing platform.

**Before launching any production site, you should:**

1. **Conduct a threat model assessment** - Understand what you're protecting, who might attack it, and what the impact would be if compromised
2. **Perform a professional security audit** - Have a qualified security professional review your Drupal configuration, infrastructure, and code
3. **Review compliance requirements** - Understand what security standards apply to your site (PCI-DSS, NHS DSP Toolkit, GDPR, ICO guidance, etc.)
4. **Establish an incident response plan** - Know what you'll do when (not if) something goes wrong
5. **Implement monitoring and logging** - You can't respond to what you don't detect

This article provides foundational hardening steps that benefit most sites. But these recommendations may be insufficient for your specific requirements, and there may be additional modules or configurations critical to your situation that this article doesn't cover.

Work with security professionals, not just articles. Your data and your users' trust depend on it.

## Resources

- [SecKit Module](https://drupal.org/project/seckit)
- [Username Enumeration Prevention Module](https://drupal.org/project/username_enumeration_prevention)
- [Password Policy Module](https://drupal.org/project/password_policy)
- [Webform Spam Words Module](https://drupal.org/project/webform_spam_words)
- [Force Users Logout Module](https://drupal.org/project/force_users_logout)
- [Admin Audit Trail Module](https://drupal.org/project/admin_audit_trail)
- [Drupal Security Advisories](https://www.drupal.org/security)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
