<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# not good enough, i need much more, it should be like a github doc but i want the initial config we did and then everything else

Here is an extremely detailed, well-structured documentation-style guide covering your entire Docker Mailserver setup process, including initial configuration, all challenges encountered, and how each was solved. This is modeled as a comprehensive GitHub-style README documentation:

***

# Docker Mailserver Setup and Troubleshooting Guide for mail.raghavvram.online

## Overview

This document details the full lifecycle of setting up a self-hosted mail server using the official `docker-mailserver` container image. It covers initial environment and Docker configurations, DNS and SSL integration, custom port management for a rootless Docker setup, enabling SMTP authentication, firewall rules, troubleshooting unexpected runtime errors, and achieving a stable secure mail service capable of sending and receiving authenticated email with clients like Thunderbird.

***

## Table of Contents

- [1. Prerequisites](#1-prerequisites)
- [2. Initial Environment Setup](#2-initial-environment-setup)
- [3. Docker Compose Configuration](#3-docker-compose-configuration)
- [4. Port Remapping for Rootless Docker](#4-port-remapping-for-rootless-docker)
- [5. SSL/TLS Certificate Mounting](#5-ssltls-certificate-mounting)
- [6. Mail User Provisioning](#6-mail-user-provisioning)
- [7. Firewall and Azure NSG Configuration](#7-firewall-and-azure-nsg-configuration)
- [8. Enabling and Troubleshooting SMTP Authentication](#8-enabling-and-troubleshooting-smtp-authentication)
- [9. Testing Mail Flow and Connectivity](#9-testing-mail-flow-and-connectivity)
- [10. Challenges Faced and Resolutions](#10-challenges-faced-and-resolutions)
- [11. Final Recommendations](#11-final-recommendations)

***

## 1. Prerequisites

- Ubuntu/Debian-based Azure VM or equivalent server
- Rootless Docker installed and running
- Domain `raghavvram.online` with DNS control for MX, SPF, DKIM, DMARC records
- Let's Encrypt SSL certificates for `mail.raghavvram.online`
- Basic Docker and networking familiarity

***

## 2. Initial Environment Setup

- Create persistent directories on host for mail data and configs:

```bash
mkdir -p ./docker-data/mail-data ./docker-data/mail-state ./docker-data/mail-logs ./docker-data/config ./letsencrypt
```

- Place your LetsEncrypt certs inside `./letsencrypt` folder matching `/etc/letsencrypt/live/mail.raghavvram.online` structure.

***

## 3. Docker Compose Configuration

```yaml
version: "3.8"

services:
  mailserver:
    image: ghcr.io/docker-mailserver/docker-mailserver:latest
    container_name: mailserver
    hostname: mail.raghavvram.online
    env_file: mailserver.env
    ports:
      - "2525:25"      # SMTP remapped
      - "1143:143"     # IMAP remapped
      - "465:465"      # SMTPS
      - "1587:587"     # Submission remapped
      - "1993:993"     # IMAPS remapped
    volumes:
      - ./docker-data/mail-data:/var/mail
      - ./docker-data/mail-state:/var/mail-state
      - ./docker-data/mail-logs:/var/log/mail
      - ./docker-data/config:/tmp/docker-mailserver/
      - ./letsencrypt:/etc/letsencrypt
      - /etc/localtime:/etc/localtime:ro
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_RESOURCE
    security_opt:
      - seccomp:unconfined
      - apparmor:unconfined
    dns:
      - 1.1.1.1
    restart: always
```


***

## 4. Port Remapping for Rootless Docker

Rootless Docker cannot bind privileged ports (<1024). To work around this:

- Remap ports below 1024 to higher available host ports (2525 for SMTP port 25, 1143 for IMAP 143, etc.).
- Adjust mail clients accordingly.
- On host firewall and Azure NSG open these remapped ports.
- Optionally create iptables PREROUTING rules to forward standard ports to remapped ones.

***

## 5. SSL/TLS Certificate Mounting

- Use `SSL_TYPE=letsencrypt` in your environment (`mailserver.env`) for automatic loading of mounted certs.
- Mounted certs found inside container at `/etc/letsencrypt/live/mail.raghavvram.online/`.
- Verify correct certificate files exist and permissions allow container access.

***

## 6. Mail User Provisioning

Add users inside running container with:

```bash
docker exec -it mailserver setup email add user@raghavvram.online superCool@007
```

User creation is mandatory before Dovecot (IMAP server) will start.

List users:

```bash
docker exec -it mailserver setup email list
```


***

## 7. Firewall and Azure NSG Configuration

- Open inbound TCP ports for all remapped ports (2525, 1143, 1587, 465, 1993) on VM firewall (ufw/iptables).
- Open matching inbound rules on Azure Network Security Group attached to your VM.
- Example ufw commands:

```bash
sudo ufw allow 1143/tcp
sudo ufw allow 2525/tcp
sudo ufw allow 1587/tcp
```


***

## 8. Enabling and Troubleshooting SMTP Authentication

- Ensure Postfix SASL is enabled (check with `postconf smtpd_sasl_auth_enable`).
- Set `smtpd_sasl_auth_enable = yes` if not set.
- Confirm `smtpd_recipient_restrictions` contains:

```
permit_sasl_authenticated,
permit_mynetworks,
reject_unauth_destination,
[policy checks...]
```

- Remove duplicate `check_policy_service` entries.
- Restart container to apply config changes.
- Test SMTP AUTH with `swaks` on submission port:

```bash
swaks --to user@raghavvram.online --from user@raghavvram.online --server mail.raghavvram.online --port 1587 --auth LOGIN --auth-user user@raghavvram.online --auth-password superCool@007 --tls
```


***

## 9. Testing Mail Flow and Connectivity

- Test IMAP connectivity locally and remotely:

```bash
openssl s_client -connect mail.raghavvram.online:1143 -starttls imap
```

- Test SMTP connection and AUTH with swaks as above.
- Configure Thunderbird with adjusted ports and authentication settings matching your remapped ports.
- Confirm sending and receiving emails works end-to-end.

***

## 10. Challenges Faced and How They Were Solved

| Challenge | Cause | Resolution |
| :-- | :-- | :-- |
| Container init fails on `kernel.domainname` | Docker domainname option requires system permissions | Removed `domainname` from docker-compose, rely only on hostname and env vars |
| Unable to bind privileged ports (<1024) | Rootless Docker restricts binding low ports | Remapped ports to higher ranges, updated firewall and NSG accordingly |
| Missing mail accounts causing Dovecot fail | No users were provisioned or persisted | Added mail users using `setup email add`, verified persistence and existence |
| SMTP AUTH not advertised | `smtpd_sasl_auth_enable` was disabled | Enabled SASL auth, restarted postfix, confirmed SMTP AUTH advertised |
| Duplicate policy entries in Postfix config | Config contained repeated `check_policy_service` rules | Removed duplicates, validated policy services running |
| SMTP AUTH connection lost after AUTH | Misconfigured or mismatched username/password or client settings | Verified client config, reset passwords, checked logs with enhanced verbosity |
| Network connectivity for remapped ports | Azure NSG or VM firewall blocking custom high ports | Opened inbound rules for remapped ports on NSG and firewall |


***

## 11. Final Recommendations

- Regularly back up your Docker volumes and config folders.
- Automate renewal and deployment of Let’s Encrypt certs.
- Monitor logs (`docker logs mailserver`) for errors and audit purposes.
- Harden security: enable Fail2Ban, strong passwords, and SPF/DKIM/DMARC DNS records.
- Keep Docker Mailserver image up to date.
- Test mail flow periodically using command line tools and client apps.
- Document your network and firewall configuration thoroughly for future maintenance.
- Consider adding webmail services (e.g., RainLoop, Roundcube) separately if desired.

***

This guide compiles all steps, configurations, challenge resolutions, and testing procedures to get a fully functional, secure, and maintainable self-hosted mail server running on your custom domain and infrastructure.

If you want, this can be formatted as Markdown suitable for GitHub README or extended with automation scripts and example config files.
<span style="display:none">[^1][^10][^11][^12][^13][^14][^15][^16][^17][^18][^19][^2][^20][^3][^4][^5][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://github.com/docker-mailserver/docker-mailserver

[^2]: https://docker-mailserver.github.io/docker-mailserver/latest/

[^3]: https://github.com/orgs/docker-mailserver/discussions/3965

[^4]: https://github.com/jeboehm/docker-mailserver

[^5]: https://docker-mailserver.github.io/docker-mailserver/latest/examples/tutorials/basic-installation/

[^6]: https://www.ionos.com/digitalguide/e-mail/technical-matters/set-up-docker-mail-server/

[^7]: https://github.com/docker-mailserver/docker-mailserver/issues/3648

[^8]: https://github.com/technicalguru/docker-mailserver-roundcube

[^9]: https://www.youtube.com/watch?v=kdH5eSo3_fY

[^10]: https://docker-mailserver.github.io/docker-mailserver/latest/config/debugging/

[^11]: https://flywp.com/blog/13079/self-host-email-with-docker-mailserver/

[^12]: https://blog.l3zc.com/en/2024/09/docker-mailserver-deployment-recap/

[^13]: https://forums.unraid.net/topic/122875-support-docker-mailserver/

[^14]: https://hub.docker.com/r/nowsci/docker-mailserver

[^15]: https://www.youtube.com/watch?v=NhoSOPGk3q0

[^16]: https://smtpbd.com/docker-mail-server-for-testing-step-by-step-guide/

[^17]: https://pkg.go.dev/github.com/scheidti/docker-mailserver-aliases

[^18]: https://webshanks.com/setup-docker-mailserver-on-debian-12/

[^19]: https://henrywithu.com/use-docker-mailserver-to-build-self-hosted-mail-server/

[^20]: https://github.com/tomav/docker-mailserver/issues/115

