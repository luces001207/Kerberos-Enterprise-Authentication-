# KERBERizing Services: Building a Single Sign-On Architecture with Kerberos and Docker

**Author:** En Mong (Lucas)
**Tools:** Kerberos (MIT), Docker, OpenSSH, Apache HTTP Server, Ubuntu 18.04

---

## Overview

In this project, I designed and deployed a fully functional **Kerberos Single Sign-On (SSO) architecture** using Docker containers. The goal was to simulate an enterprise environment where multiple administrators need centralized, passwordless authentication across SSH and web servers — eliminating the pain of managing credentials across every individual machine.

This is a real-world problem. Imagine an organization with 3 Linux servers and 2 admins. Every admin has a separate account on each server, and password changes must be repeated everywhere. Now scale that to 7 servers and 3+ admins — the current approach becomes error-prone and unmanageable. Kerberos solves this by centralizing authentication through a trusted third party: the **Key Distribution Center (KDC)**.

By the end of this project, I had a working architecture where a user authenticates **once** with the KDC and can then access both SSH and Apache web services — without entering a password again.

---

## Architecture

The environment consists of four Docker containers connected via a custom bridge network (`10.6.0.0/16`), simulating an isolated enterprise network:

![Kerberos Architecture Diagram](https://github.com/luces001207/Kerberos-Enterprise-Authentication-/blob/main/Kerberos%20Architecture%20Diagram.png)

| Container | Role | IP Address |
|---|---|---|
| `kerberos_server` | KDC (AS + TGS) | 10.6.0.5 |
| `kerberos_client2` | Client workstation | 10.6.0.6 |
| `ssh-server` | Kerberized SSH service | 10.6.0.4 |
| `apache-server` | Kerberized Apache web service | 10.6.0.3 |

As shown in the diagram, all four containers sit on a **Docker virtual network** running on the Docker Engine, which itself runs on the host machine. The client first authenticates with the KDC to obtain a TGT, then uses that TGT to obtain service-specific tickets for SSH and Apache — all without re-entering credentials. All containers use static IPs and DNS resolution via Docker's `extra_hosts` configuration, which is critical because Kerberos relies heavily on **hostname-based service principal resolution**.

### Why Docker Networking Matters

Kerberos authentication depends on proper DNS and hostname resolution for service principals (e.g., `host/ssh-server.kerberosprojectXXXX.com`). Docker's bridge network with static IP assignments and host entries allows us to replicate an enterprise network topology where each container behaves as an independent server on the same LAN. Without this, Kerberos ticket validation would fail because the KDC, clients, and services must all agree on hostnames.

---

## How Kerberos Works (Quick Refresher)

Kerberos is a **symmetric-key, ticket-based** authentication protocol built on the concept of a trusted third party. Here's the flow:

1. **Pre-Authentication:** The user sends credentials to the **Authentication Server (AS)**, which is part of the KDC.
2. **TGT Issuance:** If credentials are valid, the AS issues a **Ticket Granting Ticket (TGT)** — essentially a "master pass" that proves the user's identity.
3. **Service Ticket Request:** When the user wants to access a service (SSH, HTTP, etc.), they present the TGT to the **Ticket Granting Server (TGS)**, which issues a **service ticket** specific to that service.
4. **Service Authentication:** The user presents the service ticket to the target service. The service validates it using its **keytab** (a file containing the service's secret key, shared with the KDC). No password is transmitted.

The result: **authenticate once, access everything.**

---

## Implementation Walkthrough

### 1. Docker Environment Setup

I created a `docker-compose.yaml` defining all four containers with their respective Dockerfiles, static IPs, and cross-container DNS entries. Each container is built from Ubuntu 18.04 with relevant packages pre-installed.

```bash
docker-compose up -d --build
```

After composing, I verified all containers were running and performed ping tests from the client to all three servers to confirm network connectivity.

### 2. KDC Server Configuration

On the `kerberos_server` container, I installed and configured the KDC and admin server:

```bash
apt install krb5-kdc krb5-admin-server
```

The Kerberos realm was configured as `kerberosprojectXXXX.com`, with the KDC and admin server both hosted at `server.kerberosprojectXXXX.com`. I verified the configuration in `/etc/krb5kdc/kdc.conf`, which showed a **10-hour ticket lifetime** and AES256/AES128 encryption support.

### 3. Kerberos Database & Principal Creation

I created the Kerberos database and added the following principals using `kadmin.local`:

**User Principals:**
- `en@kerberosprojectXXXX.com` — Primary admin user
- `en2@kerberosprojectXXXX.com` — Secondary user for Apache ACL testing

**Service Principals (with random keys):**
- `host/ssh-server.kerberosprojectXXXX.com@kerberosprojectXXXX.com`
- `HTTP/apache.kerberosprojectXXXX.com@kerberosprojectXXXX.com`

I then generated **keytab files** for each service:

```bash
kadmin.local: ktadd -k /etc/ssh-server.keytab host/ssh-server.kerberosprojectXXXX.com
kadmin.local: ktadd -k /etc/apache.keytab HTTP/apache.kerberosprojectXXXX.com
```

These keytabs were copied from the KDC container to the host machine as an intermediary, then distributed to their respective service containers.

### 4. Client Configuration

On the client container, I installed `krb5-user` and configured `/etc/krb5.conf` to point to the correct realm and KDC server. This ensures that when the client runs `kinit`, it knows where to send the authentication request.

### 5. SSH Server — Kerberization

This was one of the more involved steps. On the SSH server container, I:

1. **Edited `/etc/krb5.conf`** — Replaced the default MIT realm configuration with our project realm and KDC address.
2. **Created a local user account (`en`)** — Kerberos authenticates the identity, but the SSH server still needs a local account to map the principal to.
3. **Configured `/etc/ssh/sshd_config`** to enable Kerberos/GSSAPI authentication:

```
KerberosAuthentication yes
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
KerberosTicketCleanup no
UsePAM yes
```

Setting `KerberosTicketCleanup` to `no` preserves the ticket on the server side, which is useful for verifying authentication in logs.

4. **Installed the keytab** — The `ssh-server.keytab` was copied into the container and renamed to `/etc/krb5.keytab` with **600 permissions** (`-rw-------`, root:root). These restrictive permissions are critical — the keytab contains the service's secret key. If compromised, an attacker could impersonate the SSH server.

### 6. Apache Server — Kerberization

The Apache setup followed a similar pattern:

1. **Configured `/etc/krb5.conf`** with the project realm.
2. **Installed `libapache2-mod-auth-kerb`** — the Apache module that enables Kerberos authentication.
3. **Placed the keytab** at `/etc/apache.keytab` with **640 permissions** (`-rw-r-----`, root:www-data). The `www-data` group ownership allows the Apache process to read the keytab for ticket validation, while preventing modifications.
4. **Configured the VirtualHost** in `/etc/apache2/sites-enabled/000-default.conf`:

```apache
<Directory "/var/www/html/test">
    AuthType Kerberos
    AuthName "KERBEROS AUTHENTICATION"
    KrbAuthRealms kerberosprojectXXXX.com
    Krb5Keytab /etc/apache.keytab
    KrbMethodNegotiate On
    KrbMethodK5Passwd Off
    Require user en2@kerberosprojectXXXX.com
</Directory>
```

The `Require user` directive restricts access to **only `en2`**, demonstrating that Kerberos authentication and authorization are separate concerns — a user can be authenticated (valid ticket) but still unauthorized (not in the ACL).

5. **Created the restricted test content:**

```bash
mkdir /var/www/html/test
echo "Restricted Access to Kerberos Principals \n <XXXX>" > /var/www/html/test/restricted.test
```

---

## Testing & Validation

### SSH Authentication (User: `en`)

From the client container:

```bash
kinit en
klist   # Verify TGT
ssh -vvv en@ssh-server.kerberosprojectXXXX.com
```

The verbose SSH output confirmed **GSSAPI authentication succeeded** — the client logged in without a password prompt. The `klist` output showed both the **TGT** and the **SSH service ticket** (`host/ssh-server.kerberosprojectXXXX.com`).

### Apache Authentication (User: `en2`)

**Unauthorized access test (user `en`):**

```bash
kinit en
curl "http://apache.kerberosprojectXXXX.com/test/restricted.test" -u : --negotiate
```

Result: **401 Unauthorized** — `en` has a valid Kerberos ticket but is not in the Apache ACL.

**Authorized access test (user `en2`):**

```bash
kdestroy          # Clear previous tickets
kinit en2
curl "http://apache.kerberosprojectXXXX.com/test/restricted.test" -u : --negotiate
```

Result: **200 OK** — The restricted content was returned successfully. The `klist` output confirmed both the TGT and the HTTP service ticket for `en2`.

This clearly demonstrates the distinction between **authentication** (proving who you are) and **authorization** (proving what you're allowed to do).

---

## Key Takeaways & Security Analysis

### Keytab Security
Keytab files are the crown jewels of a Kerberized service. They contain the service's long-term secret key, which is shared with the KDC. If an attacker obtains a keytab, they can:
- Decrypt service tickets intended for that service
- Impersonate the service entirely

**Mitigations applied in this project:**
- SSH keytab: `chmod 600` (owner-only read/write), owned by `root:root`
- Apache keytab: `chmod 640` (owner read/write, group read), owned by `root:www-data`
- In production: supplement with SELinux/AppArmor policies, regular key rotation, and physical/network security of the KDC

### Scaling the Architecture
If the organization grows to 8 administrators and 7 Apache servers, the changes are straightforward:
1. **Infrastructure (Part 1):** Add 5 new Apache containers with unique IPs, hostnames, and DNS entries.
2. **KDC (Part 3):** Create 8 new user principals and 5 new HTTP service principals with `kadmin.local`, generating individual keytabs for each.
3. **Apache Servers (Part 6):** Install Kerberos packages, configure `/etc/krb5.conf`, deploy keytabs with proper permissions, and configure the VirtualHost with appropriate ACLs.
4. **Clients (Part 4):** Ensure all admin workstations have `/etc/krb5.conf` pointing to the KDC.

No changes are needed to the KDC server itself (Part 2) beyond ensuring DNS resolution for new hostnames — the beauty of centralized authentication.

### Real-World Application: University SSO
If a University adopted Kerberos (realm: `schoolname.EDU`), the flow would look like this:
- A student logs in once via `kinit`, receiving a TGT valid for the day.
- Accessing Canvas, email, library systems, SSH labs, printing, or Wi-Fi portals would each trigger an automatic service ticket request — no re-authentication needed.
- Each service validates tickets using its own keytab.
- Revoking a student's KDC principal instantly disables access to **all** services.

This is essentially how Active Directory works in enterprise Windows environments, with Kerberos as the underlying authentication protocol.

---

## Tools & Technologies

- **Kerberos (MIT krb5)** — KDC, AS, TGS, kadmin
- **Docker & Docker Compose** — Container orchestration and network simulation
- **OpenSSH** — Kerberized SSH with GSSAPI
- **Apache HTTP Server** — Kerberized web authentication with `mod_auth_kerb`
- **Ubuntu 18.04** — Base OS for all containers

---

## Repository Structure

```
kerberos/
├── apache-server/
│   └── Dockerfile
├── client/
│   └── Dockerfile
├── server/
│   └── Dockerfile
├── ssh-server/
│   └── Dockerfile
├── images/
│   └── kerberos-architecture.png
├── docker-compose.yaml
└── README.md
```

---

## Conclusion

This project gave me hands-on experience with the full Kerberos authentication lifecycle — from KDC setup and principal management to keytab distribution and service kerberization. The distinction between authentication and authorization became very tangible when `en` could obtain a valid service ticket for Apache but was still denied access because they weren't in the ACL.

Understanding Kerberos at this depth is directly applicable to enterprise environments where Active Directory, LDAP, and SSO systems are the backbone of identity and access management. It also reinforced the importance of key management and the principle of least privilege when protecting sensitive cryptographic material like keytab files.

---
