# Search Engine Dorks Cheatsheet

## Disclaimer: Ethical and Responsible Use Only

**IMPORTANT: READ BEFORE USING**

The search queries (often called "dorks") listed in this document are powerful tools that can uncover a wide range of information on the internet. While many uses are legitimate (e.g., for security research by authorized professionals, finding publicly available documents), these techniques can also inadvertently or intentionally expose sensitive, private, or confidential information, or identify security vulnerabilities.

*   **Legality:** Accessing systems, files, or information without proper authorization is illegal in most jurisdictions. This includes using dorks to find and access private data, login pages for systems you don't own, or exploit vulnerabilities.
*   **Ethical Responsibility:** Always use this knowledge ethically and responsibly. Do not attempt to access, download, or use information that you are not authorized to view or possess. Respect privacy and data protection principles.
*   **Purpose of this Cheatsheet:** This document is for educational and informational purposes only, primarily for individuals learning about information security, search engine capabilities, and how information can be inadvertently exposed. It is NOT an encouragement to engage in unauthorized activities.
*   **Consequences:** Misuse of these techniques can lead to severe legal consequences, including civil and criminal penalties. It can also cause harm to individuals and organizations if sensitive data is exposed or systems are compromised.

**By using the information in this cheatsheet, you agree to do so responsibly and in compliance with all applicable laws and ethical guidelines. If you are unsure about the legality or ethics of a particular search or action, err on the side of caution and do not proceed.**

## Understanding Common Dork Operators

<!-- These operators help refine searches in engines like Google, Bing, DuckDuckGo, etc. Syntax might vary slightly between engines. -->

*   **`"exact phrase"`:** Searches for the exact phrase within the quotes.
    *   Example: `"confidential internal report"`
*   **`-term`:** Excludes results containing the specified term.
    *   Example: `jaguar -car` (finds the animal, not the vehicle)
*   **`OR` or `|`:** Searches for one term or another. Must be uppercase `OR`.
    *   Example: `cisco OR juniper configuration` or `cisco | juniper configuration`
*   **`AND`:** (Usually implicit) Ensures all terms are present.
    *   Example: `apache AND "log file"`
*   **`*` (Wildcard):** Acts as a placeholder for unknown words.
    *   Example: `"config files for * version"`
*   **`site:<domain.com>`:** Restricts search to a specific website or domain.
    *   Example: `site:example.com admin`
*   **`filetype:<ext>` or `ext:<ext>`:** Restricts results to specific file types.
    *   Example: `filetype:pdf annual report` or `ext:sql backup`
*   **`inurl:<text>`:** Searches for URLs containing specific text. Often used for paths or parameters.
    *   Example: `inurl:login admin` or `inurl:.php?id=`
*   **`intitle:<text>`:** Searches for pages with specific text in their HTML title.
    *   Example: `intitle:"index of" "backup"`
*   **`intext:<text>`:** Searches for specific text within the content/body of a page.
    *   Example: `intext:"password" filetype:log`
*   **`cache:<url>`:** Shows the search engine's cached version of a URL. Useful for viewing pages that may have changed or are offline.
    *   Example: `cache:example.com`
*   **`related:<url>`:** Finds web pages that are similar to a specified web page.
    *   Example: `related:example.com`
*   **`info:<url>`:** Presents some information that the search engine has about a particular URL (e.g., cache, related pages, links).
    *   Example: `info:example.com`
*   **`allinurl:<text>`:** Similar to `inurl:`, but all specified terms must appear in the URL.
    *   Example: `allinurl:admin login`
*   **`allintitle:<text>`:** Similar to `intitle:`, but all specified terms must appear in the title.
    *   Example: `allintitle:admin panel login`
*   **`allintext:<text>`:** Similar to `intext:`, but all specified terms must appear in the page content.
    *   Example: `allintext:username password "login failed"`

## Dork Categories & Examples

<!-- The following dorks are examples. Replace placeholders like `<keyword>`, `<domain.com>`, etc., with your specific search terms.
     Remember the ethical use disclaimer. -->

### Directory Listings
<!-- These dorks often uncover unintentionally exposed directories on web servers, which can reveal file structures and sensitive files. -->
```text
# Standard directory listing titles
intitle:"index of"
intitle:"index of /"
intitle:"index of" "backup"
intitle:"index of" "database"
intitle:"index of" "password"

# Variations sometimes used by servers
intitle:?index.of?
intitle:?index.of? "src"
```
<!-- Explanation: `intitle:"index of"` looks for pages with that common phrase in their HTML title, a strong indicator of a server-generated directory listing. -->

<!-- These examples demonstrate how to combine operators to find specific types of files or information. -->

### Specific Filetype Searches
<!-- Finding specific types of files can reveal documents, logs, code, backups, etc. -->
```text
# Configuration files
filetype:xml inurl:config "password"
filetype:ini "database_password"
filetype:cfg "username" "password"
filetype:env "DB_PASSWORD" OR "API_KEY"
ext:yaml "api_key" OR "secret_key"

# Log files
filetype:log "error" "debug" username
intext:"access denied" filetype:log
intext:"php error" ext:log "on line"

# Database files / Backups
filetype:sql "INSERT INTO" "password"
filetype:sql "dump" OR "backup"
filetype:mdb "admin" "users" # Microsoft Access DB
ext:dbf "contacts" # dBase file

# Backup files
filetype:bak "database" "backup"
ext:bkf "backup" OR "full"
ext:bkp site:example.com
ext:zip "backup" inurl:backup OR inurl:archive
ext:tar.gz "backup" "full_system"

# Sensitive documents
filetype:pdf "confidential" "internal use only"
filetype:doc "meeting minutes" "secret project"
filetype:xls "employee list" "salary" OR "ssn"

# Source code files (can reveal credentials or logic flaws)
filetype:php intext:"mysqli_connect" "password"
filetype:java inurl:src "private key" OR "AWS_SECRET_ACCESS_KEY"
filetype:py "SECRET_KEY =" OR "SQLALCHEMY_DATABASE_URI ="
```
<!-- Explanation: `filetype:` or `ext:` restricts results to the specified extension. Combining with keywords in `intext:` or `inurl:` narrows the search for sensitive content within those files. -->

### Login Pages & Admin Portals
<!-- Finding login pages can be a first step in security assessments (for authorized testing) or for users trying to find a legitimate login page. -->
```text
intitle:"Login" inurl:admin OR inurl:login
intitle:"Admin Control Panel" OR intitle:"Administration"
inurl:/admin/login.php OR inurl:/admin/index.html
inurl:/wp-admin/ OR inurl:/wp-login.php
intitle:"phpMyAdmin" "Welcome to phpMyAdmin"
intitle:"Router Admin" "Password" OR intitle:"Router Login"
intitle:"Dashboard" inurl:admin
```
<!-- Explanation: Looks for common titles and URL components associated with administrative interfaces and login forms. -->

### Vulnerability Information & Error Messages
<!-- Specific error messages or version numbers can indicate potentially vulnerable systems. -->
<!-- This is for identifying *potential* vulnerabilities during authorized security assessments or for developers to find their own exposed errors.
     Actual exploitation of vulnerabilities is illegal without authorization. -->
```text
# SQL errors (may indicate SQL injection susceptibility)
intext:"SQL syntax error near"
intext:"mysql_fetch_array()" "Warning"
intext:"unclosed quotation mark after the character string"

# PHP errors/info (may reveal path disclosure or configuration issues)
intext:"PHP Parse error" filetype:php
intext:"PHP Warning: include(" filetype:php
filetype:php "phpinfo()" "PHP Version"

# Specific software versions (can be searched for known vulnerabilities)
intitle:"index of" "Apache v2.2.3"
"PHP version 5.6.0" "powered by"
"WordPress 4.7.0" "Just another WordPress site"
"Joomla! 3.6" "Administration Login"

# Exposed Git or SVN directories / files
inurl:/.git/config # Exposes git configuration, potentially repo URL
inurl:/.svn/wc.db # Exposes SVN metadata
intitle:"index of" ".git"
intitle:"index of" ".svn"
```
<!-- Explanation: Searching for common error messages or specific software versions (especially older ones) can help identify systems that might be misconfigured or unpatched. -->

### Information Disclosure
<!-- Dorks that might reveal email addresses, internal path disclosures, API keys, or other potentially sensitive data. -->
```text
# Email lists
filetype:xls inurl:"email.xls" OR inurl:"contacts.xls"
filetype:csv "email" "contact list"
allintext:email name "@example.com" filetype:csv

# Path disclosures in error messages
intext:"Warning: include_path="
intext:"Fatal error: require_once() [function.require]: Failed opening required"

# API keys or tokens, especially in code repositories
# Use with `site:github.com`, `site:gitlab.com`, `site:pastebin.com` etc.
site:github.com "Authorization: Bearer sk_live_" # Stripe live API keys
site:github.com "xoxp-" OR "xoxb-" # Slack tokens
site:pastebin.com "BEGIN RSA PRIVATE KEY"
"aws_access_key_id" "aws_secret_access_key" filetype:yml OR filetype:json
```
<!-- Explanation: These look for common patterns of inadvertent information exposure. Searching code platforms or paste sites for leaked credentials is a common use. -->

### Site-Specific Searches
<!-- Combining `site:` with other operators to deeply search a specific domain for various types of information. -->
```text
site:example.com filetype:pdf "internal" OR "proprietary"
site:example.com inurl:admin -inurl:help -inurl:support # Try to find admin panels, exclude help pages
site:example.com intitle:"index of" "backup"
site:example.com intext:"password" OR intext:"secret" filetype:log OR filetype:txt
site:example.com ext:sql OR ext:db OR ext:sqlite
```
<!-- Explanation: `site:example.com` restricts all other search terms to only that domain, allowing for focused reconnaissance or information retrieval. -->

### IoT / Networked Devices
<!-- Finding exposed Internet of Things (IoT) devices or other networked systems like printers, cameras, routers. -->
<!-- WARNING: Accessing or attempting to log into these devices without explicit authorization is illegal and unethical. For research and authorized testing only. -->
```text
inurl:Login.html intitle:"Network Camera" OR intitle:"IP Camera"
intitle:"router setup" inurl:config.html OR inurl:settings.php
inurl:/cgi-bin/luci OR inurl:/cgi-bin/admin # Common for OpenWrt and other embedded devices
intitle:"D-Link Router" "Login" OR intitle:"NETGEAR Router Login"
intitle:"Printer Status" OR intitle:"Printer Settings"
intext:"Synology DiskStation Manager" "Login"
```
<!-- Explanation: Many IoT and networked devices have default titles or URL structures for their web interfaces that can be easily searched. -->

### Example Dork Combinations (from original sheet, now categorized)

<!-- The following examples were in the original sheet and are good illustrations of combining operators. -->

#### Finding Document/Archive Files for a Specific Topic (Directory Listing)
<!-- This dork attempts to find directory listings containing PDF, RAR, ZIP, TXT, or DOC files related to "c language". -->
```text
intitle:?index.of? pdf|rar|zip|txt|doc "c language"
```

#### Finding Video Files for a Specific Show/Movie (Directory Listing)
<!-- This dork attempts to find directory listings containing MP4, MKV, AVI, or WEBM video files related to "jujutsu kaisen". -->
```text
intitle:?index.of? mp4|mkv|avi|webm "jujutsu kaisen"
```

#### Finding Audio Files with Exclusions (Directory Listing)
<!-- This dork attempts to find directory listings with MP3 or WAV audio files for "black sabbath",
     while excluding results that mention "backing-tracks" or "cover". -->
```text
intitle:?index.of? mp3|wav "black sabbath" -"backing-tracks" -cover
```

## Potentially Useful Resources (Use Responsibly)

<!-- This section lists external resources. Users should exercise caution and be aware of the nature of these sites, including potential copyright implications or the legality of accessing certain types of content. -->

*   **Library Genesis (libgen.rs, and mirrors)**: `https://libgen.rs/` (and other domain extensions like .is, .gs, .st)
    <!-- Note: Library Genesis is a large shadow library for articles and books. Accessing copyrighted material through such sites may have legal implications depending on your jurisdiction and the content being accessed. Use responsibly and be aware of copyright laws. -->
*   **Google Hacking Database (GHDB):** `https://www.exploit-db.com/google-hacking-database`
    <!-- Note: The GHDB is a collection of dorks submitted by security professionals and researchers. It's a valuable resource for understanding what kind of information can be found, but always use this knowledge ethically and legally. -->

## Further Learning

For more advanced dorking, research the specific advanced search operators for your search engine of choice (Google, Bing, DuckDuckGo, Shodan, etc.).
Understanding how search engines index content and how websites are structured can help in crafting effective dorks.
Always remember to use such knowledge ethically, legally, and responsibly.