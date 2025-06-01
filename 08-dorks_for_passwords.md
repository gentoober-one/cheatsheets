<!--
    08-dorks_for_passwords.md
    Cheatsheet for Search Engine Dorks Related to Potential Password/Credential Exposure

    Organized and Commented for Educational & Security Awareness Purposes.
    Review and use with extreme caution, adhering to all ethical and legal guidelines.
-->

## CRITICAL WARNING AND DISCLAIMER: READ BEFORE PROCEEDING

**PURPOSE OF THIS DOCUMENT: STRICTLY EDUCATIONAL AND SECURITY AWARENESS**

The search queries ("dorks") listed in this document are provided **SOLELY FOR EDUCATIONAL AND SECURITY AWARENESS PURPOSES.** The intent is to demonstrate how misconfigurations, accidental exposures, and inadequate security practices can lead to sensitive information, including potential credentials, being indexed by public search engines and becoming discoverable. This knowledge is intended to help developers, system administrators, and security professionals understand and **PREVENT** such exposures on their own systems or systems they are **EXPLICITLY AND LEGALLY AUTHORIZED** to secure.

**ILLEGAL AND UNETHICAL USE PROHIBITED:**
-   **DO NOT USE THESE QUERIES FOR ANY MALICIOUS, UNAUTHORIZED, OR ILLEGAL ACTIVITIES.**
-   **DO NOT ATTEMPT TO ACCESS, DOWNLOAD, USE, OR SHARE ANY DATA FOUND THROUGH THESE QUERIES ON SYSTEMS, NETWORKS, OR DATABASES THAT YOU DO NOT EXPLICITLY OWN OR FOR WHICH YOU DO NOT HAVE PRIOR, WRITTEN, LEGAL AUTHORIZATION TO TEST OR AUDIT.**
-   Unauthorized access to computer systems, data, or credentials is a **SERIOUS CRIME** in virtually all jurisdictions and can lead to severe penalties, including substantial fines and imprisonment.

**YOUR RESPONSIBILITY:**
-   You are solely responsible for your actions and any consequences that may arise from the use or misuse of this information.
-   The creators and maintainers of this document disclaim any and all liability for any misuse of the information contained herein.
-   This document should be used to improve security, not to compromise it. The focus is on **DEFENSIVE AWARENESS.**

**IF YOU DISCOVER EXPOSED SENSITIVE DATA:**
-   If you inadvertently discover sensitive data belonging to others while conducting legitimate research or testing (with authorization), **DO NOT** explore it further than necessary for identification, do not download it, do not store it, and do not share it with unauthorized parties.
-   Report the finding responsibly and confidentially to the affected organization through their official security contact, bug bounty program, or other appropriate channels.

**RECOMMENDATION FOR THIS FILE:**
Given the sensitive nature of dorks that can find passwords, it is strongly recommended to handle this file with extreme care, restrict its access, and consider if all listed dorks are necessary for your educational purposes. The focus should always be on understanding *how* data gets exposed, not on accumulating lists of dorks. Renaming this file to something like `10-security_awareness_data_exposure_vectors.md` is advised to prevent misinterpretation of its intent.

---

## Introduction: Understanding the Risk

Search engine "dorks" are specially crafted search queries that use advanced search operators to find specific information that is often not visible through normal keyword searches. While powerful for legitimate research (e.g., finding specific public documents or code examples), these techniques can also be abused to uncover sensitive data that has been inadvertently exposed on the internet. This exposure typically happens due to:

1.  **Misconfigured Servers:** Web servers or applications exposing files or directories they shouldn't.
2.  **Lack of Proper Access Controls:** Sensitive files uploaded to public spaces without restriction.
3.  **Accidental Uploads:** Committing sensitive files to public code repositories, pasting into public paste sites, or including them in backups stored insecurely.
4.  **Software Vulnerabilities:** Some software might log sensitive data in web-accessible locations.
5.  **Human Error:** People unknowingly sharing files with embedded credentials or sensitive data.

This document lists examples of dork patterns that *could* lead to the discovery of exposed credentials or files commonly containing them. The primary goal is to educate on these patterns of exposure so that individuals and organizations can take steps to prevent them. **It is NOT a toolkit for finding passwords to use.**

## Common Search Engine Operators Relevant to Data Exposure

Understanding these operators is key to recognizing how information can be found (and how to protect your own):

*   **`filetype:<ext>` or `ext:<ext>`:** Restricts results to specific file types (e.g., `filetype:log`, `ext:sql`, `filetype:cfg`, `filetype:env`, `filetype:ini`). Sensitive data is often found in these file types.
*   **`intext:"<keyword>"`:** Finds pages containing specific keywords in their main text content (e.g., `intext:"password"`, `intext:"API_KEY"`, `intext:"DB_PASSWORD"`).
*   **`inurl:<path_or_keyword>`:** Searches for URLs that contain specific text, which might indicate a certain technology, common filename, or directory path. (e.g., `inurl:wp-config`, `inurl:.git`, `inurl:backup`).
*   **`intitle:"<keyword>"`:** Searches for pages with specific keywords in their HTML title (e.g., `intitle:"index of"`, `intitle:"phpmyadmin"`).
*   **`site:<domain.com>`:** Limits the search to a specific website or domain. Can be used to assess a specific organization's exposure (with authorization) or to target specific platforms (e.g., `site:github.com`, `site:pastebin.com`).
*   **`""` (Quotes):** Searches for an exact phrase (e.g., `"database password"`, `"BEGIN RSA PRIVATE KEY"`).
*   **`-<term>` or `-site:<domain.com>`:** Excludes results containing the specified term or from a specific site (e.g., `-site:stackoverflow.com` to remove help pages).
*   **`|` (Pipe) or `OR`:** Logical OR operator (e.g., `pass|passwd|password` or `API_KEY OR SECRET_KEY`).
*   **`*` (Asterisk):** Wildcard that can match one or more words.

## Categories of Potential Credential Exposure & Illustrative Dorks

The following dorks are generalized examples illustrating *types* of exposures. They are for understanding how data leakage can occur and for defensive scanning of **your own authorized assets.** Specific, highly effective dorks for direct credential harvesting are intentionally avoided or heavily generalized.

### 1. Exposed Configuration Files
<!-- Configuration files often contain database credentials, API keys, secret keys, and other sensitive settings.
     If web servers are misconfigured to serve these files as text, or if they are included in public code repositories,
     they become a major vulnerability. -->
**Risk:** Direct exposure of service credentials, API keys, encryption keys.
**Prevention:** Restrict access to config files, use environment variables for secrets, do not commit secrets to version control, use `.gitignore`.

**Illustrative Dorks (Generalized):**
```text
# Common config file extensions with keywords for passwords or keys
filetype:env "DB_PASSWORD" OR "API_KEY" OR "SECRET_KEY" OR "MAIL_PASSWORD" OR "REDIS_PASSWORD"
filetype:ini "password" OR "api_key" OR "secret" OR "db_pass"
filetype:xml "password" OR "key" OR "token" inurl:config
filetype:cfg "password" OR "secret" OR "pwd"
filetype:yml OR filetype:yaml "password" OR "api_key" OR "token" OR "secret_key_base"
filetype:json intext:"password" OR intext:"token" OR intext:"api_key" inurl:config -site:github.com -site:gitlab.com
filetype:properties "password" OR "secret" OR "db.password" OR "proxyPassword"
filetype:conf "password" OR "secret" OR "service_password" OR "rcon_password"

# Patterns for specific application configurations (generalized)
intext:"DB_PASSWORD" "define(" ext:php # Looks for PHP defines, could be WordPress, Joomla, etc.
intext:"DATABASE_URL" ext:env # Common in various frameworks like Django, Ruby on Rails
intext:"[database]" "password =" ext:ini # Generic database section in INI files
intext:"applicationSettings" "password" filetype:config # .NET application settings
intext:"<hibernate-configuration>" "<property name=\"hibernate.connection.password\">" ext:xml # Hibernate config
intext:"<dataSource>" "<property name=\"password\" value=" ext:xml # Spring dataSource config
inurl:sftp-config.json # Common file name for SFTP credentials
inurl:recentservers.xml "Pass" # FileZilla recent connections (generalized)
filetype:txt "enable secret" OR "enable password" # Network device configs
filetype:txt "UserPassword" OR "GroupPwd" # VPN client profiles (e.g. Cisco PCF)
filetype:txt "$9$" OR "$1$" # Hashed passwords in specific formats (e.g. JunOS, Cisco Type 7)
```
<!-- Comment: These dorks look for common configuration files that, if exposed, might contain plaintext or weakly protected credentials. Always ensure such files are not web-accessible and that secrets are not hardcoded. -->

### 2. Credentials in Log Files
<!-- Applications or systems sometimes incorrectly log sensitive information, including usernames, passwords, session tokens, or API keys, especially in debug mode or due to errors.
     If these log files are web-accessible, they pose a significant risk. -->
**Risk:** Exposure of user credentials, session tokens, operational data, API keys.
**Prevention:** Implement proper logging practices (avoid logging sensitive data), sanitize logs, ensure logs are not web-accessible, regularly review log content and access permissions.

**Illustrative Dorks (Generalized):**
```text
filetype:log "password" OR "pwd" OR "secret" OR "token" OR "API_KEY"
filetype:log "username=" AND "password="
intext:"login successful" "password is" ext:log
intext:"API request" ("key=" OR "token=") ext:log
intext:"debug" "user_password" filetype:log
intext:"PuTTY log" "password" ext:log # PuTTY session logs
intext:"GET http://" "password" inurl:log ext:txt # Passwords in GET request parameters logged
intext:"authentication failure" "password" filetype:log
```
<!-- Comment: These dorks search for log files containing keywords related to credentials or sensitive operations. Logging passwords or full API keys is a security anti-pattern. -->

### 3. Database Dumps or SQL Files
<!-- SQL files, especially database dumps or backups, can contain table structures, user data, including usernames and hashed (or sometimes plaintext) passwords.
     If these files are publicly accessible, it's a massive data breach waiting to happen. -->
**Risk:** Full database exposure, including user tables, password hashes, personal information, application data.
**Prevention:** Never expose database dumps publicly, secure database backup locations, encrypt sensitive backups, use strong hashing for passwords in the database.

**Illustrative Dorks (Generalized):**
```text
filetype:sql "CREATE TABLE" ("password" OR "passwd" OR "user_secret")
filetype:sql "INSERT INTO" ("users" OR "auth" OR "members" OR "customers") ("password" OR "pass_hash")
ext:sql "SQL DUMP" "username" "password"
ext:sql.gz OR ext:sql.zip OR ext:sql.tar.gz "database backup" OR "db_dump"
filetype:sql "IDENTIFIED BY" OR "ALTER USER" # SQL commands that might contain passwords
```
<!-- Comment: Searches for SQL files, particularly those that might be database dumps or contain user table creation/insertion statements with password fields. Hashed passwords are still a risk if weak algorithms are used or if they are exposed alongside user data. -->

### 4. Exposed Backup Files
<!-- Backups of websites, configurations, databases, or specific sensitive files, if not properly secured, can be found by search engines.
     These often contain the same sensitive information as the live files, sometimes with less stringent access controls. -->
**Risk:** Access to source code, configurations, credentials, historical data, private keys.
**Prevention:** Secure backup locations (not web-accessible), restrict access, encrypt sensitive backups, ensure backup files don't use predictable names or common extensions in web roots.

**Illustrative Dorks (Generalized):**
```text
filetype:bak OR filetype:bkp OR filetype:old OR filetype:sav OR ext:php~
inurl:backup OR inurl:archive OR inurl:dump
intitle:"index of" "backup" OR "archive" OR "dumps"
ext:zip OR ext:tar.gz "config_backup" OR "db_backup" OR "site_archive"
inurl:wp-config.bak OR inurl:wp-config.php.old # Specific backup names for WordPress
filetype:bak inurl:passwd OR inurl:shadow OR inurl:htusers # Backups of authentication files
```
<!-- Comment: Looks for common backup file extensions or backup-related terms in URLs and titles. Backups of sensitive files (like wp-config.php or .htpasswd) are particularly risky. -->

### 5. Credentials in Plaintext Documents or Spreadsheets
<!-- Users sometimes store credentials in plaintext documents, spreadsheets, or text files for convenience.
     If these are accidentally uploaded to web servers, shared drives indexed by search engines, or committed to version control, credentials can be exposed. -->
**Risk:** Direct exposure of usernames, passwords, API keys, license keys.
**Prevention:** Educate users on secure password management (use password managers), avoid storing credentials in plaintext files, implement DLP (Data Loss Prevention) policies.

**Illustrative Dorks (Generalized):**
```text
filetype:xls OR filetype:xlsx OR filetype:csv "username" "password" "login" "email"
filetype:doc OR filetype:docx "password list" OR "credentials" OR "account details"
filetype:txt "ssh password" OR "ftp login" OR "server credentials" OR "passlist"
intext:"Default password is:" OR "Your password is:" filetype:pdf OR filetype:doc OR filetype:txt # Default/temporary credentials in manuals or instructions
"passwords.xlsx" OR "credentials.txt" OR "logins.csv"
```
<!-- Comment: These search for office documents or text files containing common keywords related to storing lists of credentials. "Default password is" is common in product manuals. -->

### 6. Credentials Exposed in Public Code Repositories & Paste Sites
<!-- Developers might accidentally commit API keys, passwords, private keys, or other secrets to public code repositories (GitHub, GitLab) or paste them on sites like Pastebin, Gist, etc. -->
**Risk:** Compromise of developer accounts, service accounts, application infrastructure, cloud services, third-party APIs.
**Prevention:** Use `.gitignore` for sensitive files, use environment variables or dedicated secret management systems (like HashiCorp Vault, AWS Secrets Manager), implement pre-commit hooks to scan for secrets, regularly audit public repositories and paste sites for accidental leaks.

**Illustrative Dorks (Targeting specific platforms):**
```text
site:github.com OR site:gitlab.com "SECRET_KEY" OR "API_KEY" OR "token" OR "password" (filename:.env OR filename:config.py OR filename:settings.yml OR filename:docker-compose.yml)
site:pastebin.com OR site:gist.github.com "BEGIN RSA PRIVATE KEY" OR "password =" "username =" OR "aws_access_key_id"
site:s3.amazonaws.com filetype:xls OR filetype:csv "password" OR "key" # Misconfigured S3 buckets
site:trello.com intext:password OR intext:api_key # Public Trello boards
# For specific credential types:
site:github.com "sk_live_" # Stripe live API keys
site:github.com "xoxp-" OR "xoxb-" # Slack tokens
```
<!-- Comment: These dorks use the `site:` operator to target platforms where code, configurations, and text snippets are commonly shared. They look for keywords indicative of credentials or sensitive keys. Automated tools are often used by attackers (and defenders) to scan these sites for leaks. -->

### 7. Service-Specific Files or Default Credential Exposure
<!-- Some applications or devices have well-known configuration files, default paths, or default credentials that can be searched for if exposed. -->
**Risk:** Unauthorized access using default or easily guessable credentials, exploitation of known vulnerabilities in specific software versions if admin interfaces are exposed.
**Prevention:** Always change default credentials immediately upon setup, keep software and firmware updated, restrict access to administrative interfaces (e.g., via firewall or IP whitelisting).

**Illustrative Dorks (Generalized Examples - specific app dorks can be very risky):**
```text
intitle:"phpMyAdmin" "Default username is" OR "Default login"
inurl:"/xampp/" "security.html" OR "defaultusers.txt" # Older XAMPP setups
filetype:pcf "GroupPwd" # Cisco VPN profiles (often contain hashed group passwords)
intitle:"index of" "CCCam.cfg" # CCcam config file, may contain credentials
inurl:proftpdpasswd # Exposed proftpd password files (if misconfigured)
inurl:ventrilo_srv.ini "AdminPassword" # Ventrilo server config
inurl:server.cfg "rcon_password" # Game server remote console password
intext:"enable password 7" OR intext:"enable secret 5" # Cisco encrypted passwords in configs
filetype:ica "[WFClient]" "Password=" # Citrix ICA files
intitle:phpinfo "mysql.default_password" # phpinfo() output exposing default MySQL password
```
<!-- Comment: These examples are more generic. Many highly specific dorks exist for finding default credentials or known vulnerable configurations in specific versions of routers, IoT devices, CMS plugins, etc. Listing those directly is beyond the scope of an educational awareness document and veers into providing attack tools. The key takeaway is that default settings are a major risk if not changed and secured. -->

### 8. Error Messages Revealing Sensitive Information
<!-- Verbose error messages can sometimes leak database passwords, API keys, internal paths, or other sensitive configuration details. -->
**Risk:** Disclosure of credentials, paths, or system internals that can aid an attacker.
**Prevention:** Configure applications for production mode (disable debug messages), use custom error pages, sanitize error output.
```text
intext:"ODBC DSN" "password=" "Microsoft OLE DB Provider for SQL Server"
intext:"Warning: mysql_connect(): Access denied for user" "using password (YES)"
intext:"Fatal error: Call to undefined function" "db_connect()" "password"
"whoops! there was an error." "db_password" OR "environment variables"
```
<!-- Comment: These dorks look for common error message patterns that might include database connection strings or other secrets. -->

---
**Final Review Note:** This document has been curated to focus on *educating about types of exposures* and how they occur, rather than providing a direct "how-to" for finding passwords. The dorks are illustrative and generalized. Always prioritize ethical considerations and legal compliance. The most effective use of this information is for **DEFENSIVE** purposes: scanning your own assets (with authorization) to identify and remediate these common types of data leakage.
