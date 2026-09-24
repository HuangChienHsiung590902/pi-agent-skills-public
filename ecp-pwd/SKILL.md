---
name: ecp-pwd
description: Reset a Chainsea ECP (aipower) account password directly in the database (bypassing the login / forgot-password flow). Use when administrator or any account is locked out, the password is unknown/forgotten, or login keeps failing. Writes a correct PasswordUtil hash into TsAccount and clears the lock counter.
---

# ECP password reset (aipower)

Reset an ECP account's password by writing the hash directly into the database.

## When to use

- `administrator` (or any user) password is forgotten / unknown.
- Account is locked after too many failed attempts.
- The shipped default `administrator` / `111111` does not work (password was changed during init).

## How ECP login works (why this is safe & correct)

1. Frontend (`quicksilver/page/user/Login.js`) fetches an RSA public key via
   `POST /aipower/Qs.Misc.getLoginPublicKey.data`, then sends
   `password = JSEncrypt.encrypt(plaintext)` — **plain password, RSA-encrypted, no extra hashing**.
2. Backend decrypts to plaintext and calls
   `AccountServiceImpl.getItem(ctx, loginName, plaintext)` →
   `PasswordUtil.matches(plaintext, storedHash)`.
3. Stored value = `PasswordUtil.encode(plaintext)`.

So resetting = writing `PasswordUtil.encode(newPlaintext)` into `default.TsAccount.FPassword`.

`PasswordUtil` (com.jeedsoft.common.basic.util), verified from bytecode:
StandardPasswordEncoder with `secret="secret"`, SHA-256, **1024 iterations**, 8-byte random salt.
`digest = join(salt, "secret", utf8(password))` iterated 1024× SHA-256; final =
`salt(8) + digest(32)` = 40 bytes, encoded as custom Base16 mapping nibble 0..15 → `'a'..'p'`
(high nibble first). The bundled script reproduces this exactly.

## Procedure

1. Make sure the ECP DB is running — normally because **`server.bat` is up** (the `aipower`
   webapp starts an embedded MariaDB with `--skip-grant-tables` on a random port). The standalone
   `database.bat` (port 3306) also works. The script auto-detects the port and `mariadb.exe`
   client from the running `mariadbd.exe` process.

2. Run the reset (default: set `administrator` back to `111111`):

   ```powershell
   python "C:\Users\HCH\.claude\skills\ecp-pwd\scripts/reset_password.py"
   ```

   Options:
   ```powershell
   # different password and/or account
   python ...\scripts/reset_password.py "<EXAMPLE_PASSWORD>" --login administrator
   # force a specific DB endpoint if auto-detect fails
   python ...\scripts/reset_password.py 111111 --port 3306 --client "C:\com\chainsea\apache-tomcat\temp\MariaDB4j\base\bin\mariadb.exe"
   ```

   The script: verifies the account exists → `UPDATE TsAccount SET FPassword=<hash>` →
   `DELETE FROM TsUserInputPasswordErrorCount` (clears lockout) → self-checks with `matches()`.
   It prints `self-check match : True` on success.

3. Log in at `http://127.0.0.1:<httpPort>/aipower/` (httpPort from
   `apache-tomcat\conf\server.xml`, default 22821) with the account and the new password.

## Notes

- **Account cache:** ECP may cache the account in memory. If login still fails right after a
  reset, restart `server.bat` to clear the cache, then log in.
- Tables involved: `default.TsAccount` (`FLoginName`, `FPassword`, `FEnabled`),
  `default.TsUserInputPasswordErrorCount` (failed-attempt / lock counter).
- The seed/default `administrator` password is `111111` — confirmed: the init SQL log stores
  `FPassword='<LEGACY_PWD_HASH>'` which is base64(SHA1("111111")) (legacy format).
- Avoid quotes/special shell characters in the new password (the tool builds SQL by string).
- Connecting works because the embedded mariadbd runs with `--skip-grant-tables` (no DB auth).
- Related: the `ecp-server-startup` skill (starting the DB / install patch).

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
