
| User Prompt: |
|-------------|
| I have an old wordpress site called lexshift-old and a new site called lexshift-new. The two sites both run on AWS lightsail with distinct static IP addresses. The lexshift-new site is mapped to our domain at https://lexshift.com. I'm trying to access the lexshift-old site by its static IP address, but it immediately redirects to lexshift.com (the live domain) because lexshift-old runs requests as "lexshift.com", not as the IP address. How to fix so that I can access lexshift-old by the static IP address without being rerouted to lexshift-new. |

Of course. I can help you resolve this redirection issue. The behavior you're describing is standard for WordPress, and we can correct it by updating the site's address settings on your `lexshift-old` instance.

-----

### 🛡️ 1. Safety First: Create a Backup

Before making any configuration changes, the first and most critical step is to ensure you have a reliable backup.

Please log in to your Amazon Lightsail dashboard and take a **manual snapshot** of the `lexshift-old` instance. This will serve as a complete restore point if anything goes wrong.

-----

### ❓ 2. The Cause of the Redirect

The redirection is happening because your `lexshift-old` WordPress installation has its **WordPress Address (URL)** and **Site Address (URL)** configured to `https://lexshift.com`.

When you access the site via its static IP, WordPress checks its settings, sees this mismatch, and immediately redirects you to what it believes is the correct canonical URL—in this case, the live domain, which is handled by your `lexshift-new` instance.

-----

### 🛠️ 3. How to Fix the Redirection

To fix this, we need to tell the `lexshift-old` installation that its new address is its static IP. The most precise way to do this is via the command line using WP-CLI, which is included with your Bitnami stack.

This process requires **SSH access** to the `lexshift-old` Lightsail instance.

#### Step-by-Step Instructions:

1.  **Connect via SSH:** Open a terminal and connect to your `lexshift-old` instance. You will need its static IP address and your SSH key. The command will look like this:

    ```bash
    ssh -i /path/to/your-key.pem bitnami@<LEXSHIFT_OLD_STATIC_IP>
    ```

2.  **Navigate to the WordPress Directory:** Once connected, change to the directory where WordPress is installed. On a Bitnami stack, this is the standard location:

    ```bash
    cd /opt/bitnami/wordpress
    ```

3.  **Update the Site Address:** Run the following two commands, replacing `<LEXSHIFT_OLD_STATIC_IP>` with the actual static IP address of your old server. We use `http` because an SSL certificate is not valid for an IP address.

      * Update the WordPress Address (URL):
        ```bash
        wp option update siteurl 'http://<LEXSHIFT_OLD_STATIC_IP>'
        ```
      * Update the Site Address (URL):
        ```bash
        wp option update home 'http://<LEXSHIFT_OLD_STATIC_IP>'
        ```

4.  **Verify the Changes (Optional):** You can confirm the values were updated correctly by running:

    ```bash
    wp option get siteurl
    wp option get home
    ```

    The output for both should be `http://<LEXSHIFT_OLD_STATIC_IP>`.

5.  **Test in Your Browser:** Open a new browser tab (or an incognito window to avoid caching issues) and navigate to `http://<LEXSHIFT_OLD_STATIC_IP>`. The `lexshift-old` site should now load directly without redirecting.

If you follow these steps, your `lexshift-old` site will be accessible via its IP address, allowing you to work on it independently from the live `lexshift-new` site.


| User Prompt: |
|-------------|
| Please advise:

  

\`\`\`bash

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ sudo wp option update siteurl 'http://18.221.253.166'

Success: Updated 'siteurl' option.

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ sudo wp option update home 'http://18.221.253.166'

Success: Updated 'home' option.

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ sudo wp option get siteurl                               https://lexshift.com

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ sudo wp option get home

https://lexshift.com

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ |

Excellent, the log you provided shows exactly what's happening.

The commands to update the database *did* work successfully. However, the `wp option get` commands are still showing the old `lexshift.com` URL. This almost certainly means the **Site URL** and **Home URL** are hardcoded in your `wp-config.php` file.

In a Bitnami configuration, settings defined in `wp-config.php` **override** the values stored in the database. This is the root cause of the issue.

-----

### 🛠️ How to Fix It

We just need to edit one file to resolve this. You can remain in your current SSH session.

1.  **Backup the Configuration File:** Before editing, it's always a good idea to make a quick backup of the config file. Run this command from your current directory (`/opt/bitnami/wordpress`):

    ```bash
    sudo cp wp-config.php wp-config.php.bak
    ```

2.  **Edit `wp-config.php`:** Open the file using a command-line editor like `nano`:

    ```bash
    sudo nano wp-config.php
    ```

3.  **Find and Modify the Lines:** Look for the lines that define `WP_HOME` and `WP_SITEURL`. They will look like this:

    ```php
    define( 'WP_HOME', 'https://lexshift.com' );
    define( 'WP_SITEURL', 'https://lexshift.com' );
    ```

4.  **Comment Out the Lines:** The best approach is to "comment out" these lines by adding two slashes (`//`) at the beginning. This will cause WordPress to ignore them and fall back to using the values you set in the database.

    ```php
    // define( 'WP_HOME', 'https://lexshift.com' );
    // define( 'WP_SITEURL', 'https://lexshift.com' );
    ```

    Alternatively, you could change the values directly, but commenting them out is cleaner.

5.  **Save and Exit:**

      * Press `Ctrl+X` to exit `nano`.
      * Press `Y` to confirm you want to save the changes.
      * Press `Enter` to confirm the filename.

-----

### ✅ 4. Verification

Now, run the `get` commands again.

```bash
sudo wp option get siteurl
sudo wp option get home
```

This time, the output should correctly display `http://18.221.253.166`. Your `lexshift-old` site should now be fully accessible via its static IP address without redirecting.


| User Prompt: |
|-------------|
| Got it. Page loads now, but no images -- they all display with the broken image icon. Please advise. |

Of course. This is expected behavior. The issue is that while we updated the main site address, the URLs for your images and other media are still pointing to the old `lexshift.com` domain because they are saved as absolute links within your page content in the database.

To fix this, we need to perform a "search-and-replace" on the database to update all instances of the old URL to the new IP address. The safest and most reliable way to do this is with WP-CLI, which correctly handles serialized data.

-----

### 🛠️ How to Fix Broken Image Links

You can remain in your SSH session to complete these steps. We'll start with a "dry run" to simulate the changes safely before applying them.

#### Step 1: Perform a "Dry Run"

This command will show you what changes will be made without actually modifying the database. It's a critical safety check.

1.  **Navigate to the WordPress Directory:** If you're not already there, go to the correct folder.

    ```bash
    cd /opt/bitami/wordpress
    ```

2.  **Run the Search-and-Replace Dry Run:**

    ```bash
    sudo wp search-replace 'https://lexshift.com' 'http://18.221.253.166' --all-tables --dry-run
    ```

You will see a table showing how many replacements would be made in each database table. Review this to ensure it looks correct.

-----

### ✅ Step 2: Perform the Live Search-and-Replace

Once you have confirmed the dry run looks good, run the same command without the `--dry-run` flag to permanently apply the changes.

1.  **Run the Live Command:**
    ```bash
    sudo wp search-replace 'https://lexshift.com' 'http://18.221.253.166' --all-tables
    ```
    This command will go through every table in your database and update the URLs.

-----

### 🌐 Step 3: Verify the Fix

1.  **Clear Server Cache:** Your site uses **W3 Total Cache**. It's a good practice to flush its cache after a major change like this.
    ```bash
    sudo wp cache flush
    ```
2.  **Clear Browser Cache:** Open your `lexshift-old` site in a new incognito browser window or clear your browser's cache completely to ensure you are not seeing a cached version of the page.

Your images should now load correctly on the site.


| User Prompt: |
|-------------|
| Before running the live search-and-replace, I've already snapshotted the lightsail instance, and I want to run a backup from the wp-admin console as well. I'm trying to log into the old site using my prior username/password, but getting an error that the username/password was incorrect. Please help me troubleshoot getting to the wp-admin dashboard. I am connected to the site through putty. |

Of course. It's common to run into login issues after changing a site's URL, usually due to browser cookies or security plugin configurations. Since you are already connected via PuTTY, we can troubleshoot and resolve this efficiently.

Here are the most likely causes and how to fix them, starting with the simplest solution.

-----

### 1\. Rule Out Browser Issues (Easiest First)

The most common cause is that your browser is still trying to use old cookies from `lexshift.com` that are not valid for the new IP address.

  * **Action:** Open a **new incognito or private browsing window**.
  * **Navigate to:** `http://18.221.253.166/admin`
  * **Why:** This bypasses your browser's cache and cookies, ensuring you are starting a completely fresh login session with the server.

If you can log in this way, the problem was simply cached data in your browser.

-----

### 2\. Reset Your Password via Command Line (Most Direct Fix)

If an incognito window doesn't work, we can definitively rule out a credential issue by setting a new password directly using WP-CLI.

1.  **List WordPress Users:** First, let's confirm the exact username. In your PuTTY terminal, run:

    ```bash
    sudo wp user list
    ```

    This will show a table of all users. Note the exact value in the `user_login` column for your account.

2.  **Update the Password:** Now, use the following command to set a new password. Replace `USERNAME` with your actual user login and choose a new, strong password.

    ```bash
    sudo wp user update USERNAME --user_pass="YourNewStrongPasswordHere"
    ```

    You will see a "Success" message once it's done. Now, try logging in again using the new password (in an incognito window).

-----

### 3\. Temporarily Disable Plugins

If you still cannot log in after a password reset, a security plugin (like **Wordfence**, which is active on your site) may be blocking the login due to the URL change. The quickest way to check this is to temporarily disable all plugins.

1.  **Rename the Plugins Folder:** This action safely deactivates all plugins without changing any settings.

    ```bash
    sudo mv /opt/bitnami/wordpress/wp-content/plugins /opt/bitnami/wordpress/wp-content/plugins_disabled
    ```

2.  **Attempt to Log In:** Try logging in again at `http://18.221.253.166/admin`.

3.  **Important: Rename the Folder Back:** Whether the login works or not, you **must** rename the folder back to re-enable your plugins.

    ```bash
    sudo mv /opt/bitnami/wordpress/wp-content/plugins_disabled /opt/bitnami/wordpress/wp-content/plugins
    ```

If you were able to log in with plugins disabled, you can then log in to the dashboard, go to the Plugins section, and reactivate them one by one to find the culprit.

Please try these steps in order. The browser or password reset methods are the most likely to solve the problem quickly.


| User Prompt: |
|-------------|
| On step 1, when I go to the address and put in my username and password, the first thing I get is a warning screen in chrome incognito that says "3.12.34.239 doesn't support a secure connection with HTTPS". If I click "continue to site", nothing happens. Please advise. |

You've run into a classic SSL/HTTPS issue. The problem is that your browser is trying to create a secure `https://` connection, but SSL certificates are issued for domain names (`lexshift.com`), not IP addresses.

You must manually tell the browser to use an insecure `http://` connection.

-----

### 💡 A Quick Note on IP Addresses

Before we proceed, I noticed you used the IP address `3.12.34.239`. In our previous steps, we were working with `18.221.253.166` for the `lexshift-old` site. Please double-check and use the **correct static IP for your `lexshift-old` Lightsail instance** in the steps below.

-----

### ✅ Solution 1: Manually Type `http://`

This is the most direct solution and will likely solve the issue immediately.

1.  Open a new incognito window in Chrome.
2.  In the address bar, **manually type the entire address starting with `http://`**. Do not let the browser autofill or default to `https://`.
      * Type exactly: `http://18.221.253.166/admin` (or whichever IP is correct).

This forces the browser to make a direct, insecure connection, which should bypass the warning screen and allow the login page to load.

-----

### 🛠️ Solution 2: If You Are Still Being Redirected

If manually typing `http://` still results in a redirect to `https://` (and the same error), it means your server itself is configured to force all traffic to HTTPS. We need to disable that rule.

This requires SSH access, which you already have through PuTTY.

1.  **Navigate to the Apache Config Directory:**

    ```bash
    cd /opt/bitnami/apache2/conf/
    ```

2.  **Backup the Configuration File (Safety First):**

    ```bash
    sudo cp bitnami/bitnami.conf bitnami/bitnami.conf.bak
    ```

3.  **Edit the Configuration File:** Open the file in the `nano` text editor.

    ```bash
    sudo nano bitnami/bitnami.conf
    ```

4.  **Comment Out the HTTPS Redirect Rules:** Look for a block of code similar to the one below. Add a `#` at the beginning of the `RewriteCond` and `RewriteRule` lines to disable them.

      * **Find this block:**

        ```apache
        RewriteEngine On
        RewriteCond %{HTTPS} !=on
        RewriteRule ^/(.*) https://%{SERVER_NAME}/$1 [R,L]
        ```

      * **Change it to this:**

        ```apache
        RewriteEngine On
        # RewriteCond %{HTTPS} !=on
        # RewriteRule ^/(.*) https://%{SERVER_NAME}/$1 [R,L]
        ```

5.  **Save and Exit `nano`:**

      * Press `Ctrl+X`.
      * Press `Y` to confirm you want to save.
      * Press `Enter` to save the file.

6.  **Restart the Apache Server:** This step is critical for the changes to take effect.

    ```bash
    sudo /opt/bitnami/ctlscript.sh restart apache
    ```

After Apache restarts, try **Solution 1** again. The server will no longer force the redirect, and you should be able to access the login page over `http://`.


| User Prompt: |
|-------------|
| When I type \*\*only\*\* the IP address (18.221.253.166), using either \`http://\` \*\*or\*\* \`https://\`, the page loads but still no images, even after running the database replace without the \`--dry-run\` flag. When I type the IP address followed by \`/admin\` or \`wp-admin\`, the page redirects to lexshift.com (our domain running the new site). Below is my \`bitnami.conf\` file. Please advise.

  

\`\`\`nano

\# Default Virtual Host configuration.

  

\# Let Apache know we're behind a SSL reverse proxy

SetEnvIf X-Forwarded-Proto https HTTPS=on

  

<VirtualHost \_default\_:80>

  DocumentRoot "/opt/bitnami/apache/htdocs"

  # BEGIN: Configuration for letsencrypt

  Include "/opt/bitnami/apps/letsencrypt/conf/httpd-prefix.conf"

  # END: Configuration for letsencrypt

  # BEGIN: Support domain renewal when using mod\_proxy without Location

  <IfModule mod\_proxy.c>

    ProxyPass /.well-known !

  </IfModule>

  # END: Support domain renewal when using mod\_proxy without Location

  # BEGIN: Enable HTTP to HTTPS redirection

  RewriteEngine On

  # RewriteCond %{HTTPS} !=on

  # RewriteCond %{HTTP\_HOST} !^localhost

  # RewriteCond %{HTTP\_HOST} !^\[0-9\]+.\[0-9\]+.\[0-9\]+.\[0-9\]+(:\[0-9\]+)?$

  # RewriteCond %{REQUEST\_URI} !^/\\.well-known

  # RewriteRule ^/(.\*) https://%{SERVER\_NAME}/$1 \[R,L\]

  # END: Enable HTTP to HTTPS redirection

  # BEGIN: Enable www to non-www redirection

  RewriteCond %{HTTP\_HOST} ^www\\.(.\*)$ \[NC\]

  RewriteCond %{HTTP\_HOST} !^localhost

  RewriteCond %{HTTP\_HOST} !^\[0-9\]+.\[0-9\]+.\[0-9\]+.\[0-9\]+(:\[0-9\]+)?$

  RewriteCond %{REQUEST\_URI} !^/\\.well-known

  RewriteRule ^(.\*)$ http://%1$1 \[R=permanent,L\]

  # END: Enable www to non-www redirection

  <Directory "/opt/bitnami/apache/htdocs">

    Options Indexes FollowSymLinks

    AllowOverride All

    Require all granted

  </Directory>

  

  # Error Documents

  ErrorDocument 503 /503.html

  # BEGIN: Support domain renewal when using mod\_proxy within Location

  <Location /.well-known>

    <IfModule mod\_proxy.c>

      ProxyPass !

    </IfModule>

  </Location>

  # END: Support domain renewal when using mod\_proxy within Location

</VirtualHost>

Include "/opt/bitnami/apache/conf/bitnami/bitnami-ssl.conf"

\`\`\` |

I've analyzed your `bitnami.conf` file and the behavior you're describing. The issue is two-fold: Apache isn't pointing to your WordPress installation correctly, and there's likely a conflicting rule in your WordPress `.htaccess` file causing the admin redirect.

-----

### Diagnosing the Core Problems

1.  **Incorrect DocumentRoot:** Your `bitnami.conf` file has its `DocumentRoot` set to `/opt/bitnami/apache/htdocs`. This is the default Bitnami landing page, **not** your WordPress site. This is why you see a generic page without your site's images when you visit the IP address. The server is simply not loading the WordPress application.
2.  **Redirect for `/admin`:** The redirect to `lexshift.com` when you try to access the admin area is happening at the application level. WordPress itself is enforcing its canonical URL, likely through rules in its `.htaccess` file that are overriding the server configuration.

-----

### 🛠️ How to Fix It

We will perform two steps: first, point Apache to the correct directory, and second, reset the WordPress configuration file that is causing the redirect.

#### Step 1: Correct the Apache `DocumentRoot`

You must tell Apache where your WordPress files are located.

1.  **Edit the `bitnami.conf` file:**

    ```bash
    sudo nano /opt/bitnami/apache2/conf/bitnami/bitnami.conf
    ```

2.  **Change the `DocumentRoot` and `<Directory>` paths:**

      * **Find these lines:**
        ```apache
        DocumentRoot "/opt/bitnami/apache/htdocs"
        <Directory "/opt/bitnami/apache/htdocs">
        ```
      * **Change them to point to your WordPress folder:**
        ```apache
        DocumentRoot "/opt/bitnami/wordpress"
        <Directory "/opt/bitnami/wordpress">
        ```

3.  **Save and Exit `nano`** (Ctrl+X, then Y, then Enter).

4.  **Restart Apache:** This is a critical step to apply the new configuration.

    ```bash
    sudo /opt/bitnami/ctlscript.sh restart apache
    ```

After this step, visiting `http://18.221.253.166` should load your actual WordPress homepage. The images might still be broken, but we'll address that next.

-----

#### Step 2: Reset the `.htaccess` File

This file contains redirection rules generated by WordPress. We will back it up and let WordPress create a new, correct one.

1.  **Navigate to your WordPress directory:**

    ```bash
    cd /opt/bitnami/wordpress
    ```

2.  **Rename the current `.htaccess` file:** This acts as a safe backup and deactivates the old rules.

    ```bash
    sudo mv .htaccess .htaccess.bak
    ```

    *(Note: If it says the file doesn't exist, that's okay—just proceed to the next step).*

3.  **Log in to the WordPress Admin:**

      * In an incognito browser window, go to `http://18.221.253.166/admin`. It should now present you with the login screen without redirecting.
      * Log in with your credentials.

4.  **Regenerate the `.htaccess` file:**

      * In the admin dashboard, navigate to **Settings** -\> **Permalinks**.
      * You do **not** need to change any settings. Simply scroll down and click the **Save Changes** button.
      * This action will cause WordPress to automatically generate a new, clean `.htaccess` file with the correct rules for `http://18.221.253.166`.

-----

### ✅ Final Verification

After completing both steps, clear your browser cache one more time (or use a new incognito window).

  * Navigating to `http://18.221.253.166` should now load your full WordPress site with all images working correctly.
  * Navigating to `http://18.221.253.166/admin` should take you directly to the login page without any redirection to `lexshift.com`.


| User Prompt: |
|-------------|
| Great. Now I get to the wp-admin page without being redirected, but still no images on the site. Please advise. |

Excellent, getting access to the admin dashboard is the most important step. We're on the final part of the fix.

The broken images persist because the URLs pointing to them are still saved as `https://lexshift.com` within your page content. While the `wp search-replace` command is powerful, sites built with **Elementor** often require an additional step because of how it stores data.

The most likely causes are either persistent caching or URLs embedded in Elementor's files that the standard search-and-replace missed.

-----

### 1\. Clear All Caches

First, let's ensure we aren't looking at old, cached versions of the site.

  * **Server Cache:** Since you are in the terminal, run this command to flush the W3 Total Cache.
    ```bash
    sudo wp cache flush
    ```
  * **Browser Cache:** In your browser, perform a "hard refresh" by pressing **`Ctrl+Shift+R`** (or **`Cmd+Shift+R`** on a Mac). It's even better to use a new incognito window to be certain.

If the images are still broken after clearing the caches, proceed to the next step, which is the definitive fix for Elementor sites.

-----

### 2\. Use Elementor's Built-in URL Tool

This tool is specifically designed to update URLs within Elementor's complex data structures. This task is completed within the WordPress Admin dashboard.

1.  **Navigate to the Tool:** In the left-hand menu of your WordPress dashboard, go to **Elementor** -\> **Tools**.

2.  **Select the "Replace URL" Tab:** At the top of the Tools page, click on the **Replace URL** tab.

3.  **Enter the URLs:** You will see two fields. It is **critical** to enter these exactly as shown, without any typos or trailing slashes.

      * In the `Old URL` field, enter: `https://lexshift.com`
      * In the `New URL` field, enter: `http://18.221.253.166`

4.  **Run the Replacement:** Click the **"Replace URL"** button. Elementor will scan your database and update the URLs. It will show you how many instances it found and replaced.

5.  **Regenerate CSS:** After the replacement is complete, go to the **General** tab on the same page and click **"Regenerate Files & Data"**. This ensures all your styles are updated correctly.

After completing these steps, open the site in a new incognito window. Your images and site styling should now load correctly.


| User Prompt: |
|-------------|
| Excellent through step 4. At step 5, the "General" tab only has four options:

\- Elementor Cache - Clear Files & Data

\- Sync Library

\- Safe Mode

\- Debug Bar

  

Please advise. |

You are in the right place. Elementor has updated the label for that function in newer versions.

The correct option to use is **"Elementor Cache - Clear Files & Data"**.

This performs the same action as the old "Regenerate Files & Data" button. It rebuilds Elementor's CSS files and clears its internal cache, which is the final step we need to ensure all the URL changes are reflected in your site's styling.

***

### ✅ Final Steps

1.  On the **General** tab, please click the button next to **Elementor Cache**. It may be labeled "Clear Files & Data" or similar.
2.  Once the process is complete, open your site (`http://18.221.253.166`) in a **new incognito window**.

Your site should now load correctly with all images and styles intact.


| User Prompt: |
|-------------|
| Excellent. We have successfully turned on the "lexshift-old" site and can access it properly on its static IP addess (18.221.253.166), including accessing the admin dashboard. We are also able to access the new, production site that we have set up also in Lightsail with a new static IP (3.150.250.227) that launches with our domain (\`https://lexshift.com\`). I'm able to access both in the browser (\`wp-admin\`) and through Putty. Now, I need to migrate the media library from the old site to the new site, because we have assets such as logos in email footers that are linked to the lexshift.com structure (for example, \`https://lexshift.com/wp-content/uploads/2023/10/horiz\_dark\_logo.png\`). Because the new site was migrated into a new lightsail instance, it does not have the same paths to our image assets, for example. Please advise. |

Of course. Since you need the exact file paths to work on the new site, we'll migrate the entire `uploads` directory.

The most direct and efficient way to do this is a two-step process: first, we'll copy the files directly from the old server to the new one, and second, we'll use a simple plugin on the new site to register those files into the WordPress media library. This method is much faster than downloading and re-uploading everything manually.

-----

### 🛡️ 1. Safety First: Backup Both Sites

Before we begin, it's crucial to create a complete backup of **both** your `lexshift-old` and `lexshift-new` instances.

  * **Action:** Go to your Amazon Lightsail dashboard and take a **manual snapshot** of each instance. This provides a one-click restore point if anything goes wrong.

-----

### 🛠️ 2. Transfer Media Files from Old to New Server

We will use the `scp` (Secure Copy) command to copy the entire `uploads` folder over the network. You will run this command from your **local computer's terminal** (like PowerShell on Windows or Terminal on macOS), **not** from within a PuTTY session.

1.  **Open Your Local Terminal:** Make sure you are in the directory where your `.pem` key file is stored.

2.  **Run the SCP Command:** Execute the following command. Be sure to replace `<PATH_TO_YOUR_PEM_KEY>` with the actual path to your key file.

    ```bash
    scp -i <PATH_TO_YOUR_PEM_KEY> -r bitnami@18.221.253.166:/opt/bitnami/wordpress/wp-content/uploads bitnami@3.150.250.227:/opt/bitnami/wordpress/wp-content/
    ```

      * **`scp`**: The secure copy command.
      * **`-i <PATH_TO_YOUR_PEM_KEY>`**: Specifies your AWS private key for authentication.
      * **`-r`**: Stands for "recursive," which tells the command to copy the entire `uploads` folder and everything inside it.
      * **`bitnami@18.221.253.166:...`**: This is the **source**. It's the full path to the `uploads` folder on your `lexshift-old` server.
      * **`bitnami@3.150.250.227:...`**: This is the **destination**. It's the path to the `wp-content` directory on your `lexshift-new` server.

    This process may take some time depending on the size of your media library.

-----

### 🔌 3. Register Files in the New Media Library

Just copying the files isn't enough; the new WordPress site's database doesn't know they exist yet. We'll use a plugin called **Media Sync** to scan the directory and import the files into the library.

1.  **Log In:** Access the WordPress admin dashboard for your new site (`https://lexshift.com/admin`).

2.  **Install Plugin:**

      * Navigate to **Plugins** -\> **Add New**.
      * Search for **"Media Sync"**.
      * Install and **Activate** the plugin.

3.  **Run the Sync:**

      * In the left-hand menu, go to **Media** -\> **Media Sync**.
      * The plugin will automatically detect the files in your `uploads` directory that aren't yet in the media library.
      * Click the **"Sync"** or **"Scan"** button to begin the process. The plugin will then add all your transferred images and documents to the WordPress database.

-----

### ✅ 4. Final Verification and Cleanup

1.  **Check the Media Library:** Go to **Media** -\> **Library** on your new site. All the images and files from the old site should now be visible.

2.  **Test a URL:** Now, test one of your image links, like `https://lexshift.com/wp-content/uploads/2023/10/horiz_dark_logo.png`. It should now load the image correctly.

3.  **Cleanup (Optional):** Once you've confirmed everything is working, you can safely deactivate and delete the **Media Sync** plugin. Its job is done, and it's good practice to remove unused plugins.


| User Prompt: |
|-------------|
| Please advise. Perhaps should I use filezilla?

  

PS C:\\Users\\Jeffrey> scp -i "D:\\Users\\Jeffrey\\OneDrive - lexshift\\documents\\businesses\\lexshift\\Lightsail SSH Keys\\LightsailDefaultKey-us-east-2.pem" -r bitnami@18.221.253.166:/opt/bitnami/wordpress/wp-content/uploads bitnami@3.150.250.227:/opt/bitnami/wordpress/wp-content/

ssh: connect to host 3.150.250.227 port 22: Connection timed out

C:\\WINDOWS\\System32\\OpenSSH\\scp.exe: Connection closed

PS C:\\Users\\Jeffrey> |

The connection is timing out because the firewall on your **new** Lightsail instance (`lexshift-new`) is blocking the incoming connection attempt from your **old** server.

The `scp` command you're running proxies the transfer through your local machine, so the destination server sees the connection coming from the source server's IP. The quickest and most secure solution is to add a firewall rule to the new instance, specifically allowing SSH access from the old instance.

***

### ## The Recommended Fix: Adjust the Lightsail Firewall

This is the standard and most secure way to solve this. You'll add a rule to your `lexshift-new` instance to trust connections from your `lexshift-old` instance.

1.  **Navigate to your Lightsail Console:** Log in to your AWS Lightsail account.
2.  **Select the New Instance:** Click on your `lexshift-new` instance (the one with the IP `3.150.250.227`).
3.  **Go to the Networking Tab:** Click on the **"Networking"** tab for that instance.
4.  **Add a Firewall Rule:** In the "IPv4 Firewall" section, click **"Add rule"**.
    * **Application:** Select **SSH** from the dropdown list. This will automatically fill in the protocol (TCP) and port (22).
    * **Source IP address:** This is the critical step. Change the source from "Anywhere" to **"Custom"**. In the text box that appears, enter the static IP address of your **old server**: `18.221.253.166`.
    
5.  **Save the Rule:** Click the save icon or "Create" button.

The firewall will update within a minute. Once it has, run the exact same `scp` command from your PowerShell terminal. It should now connect and begin the file transfer.

***

### ## Alternative: Using FileZilla

To answer your question, yes, you can absolutely use FileZilla. This method involves downloading the files to your local machine first and then uploading them to the new server. It works perfectly well but is a two-step process.

1.  **Connect to the Old Server:**
    * In FileZilla, connect to `sftp://18.221.253.166` using the `bitnami` user and your `.pem` key.
    * Navigate to `/opt/bitnami/wordpress/wp-content/`.
    * Drag the `uploads` folder from the server pane (right side) to a folder on your local computer (left side) and wait for the download to complete.

2.  **Connect to the New Server:**
    * Disconnect from the old server.
    * Connect to `sftp://3.150.250.227` using the same credentials.
    * Navigate to `/opt/bitnami/wordpress/wp-content/` on the new server.
    * Drag the `uploads` folder from your local computer into the server's `wp-content` directory and wait for the upload to complete.

While FileZilla is a great tool, I recommend trying the firewall fix first as it's more direct for this task.


| User Prompt: |
|-------------|
| On adding firewall rule, got the following -- please advise:

  

Error

PutInstancePublicPorts\[us-east-2\]

You cannot specify a CIDR or a CIDR list alias together with "0.0.0.0/0". Specifying "0.0.0.0/0" allows all CIDRs and CIDR list aliases.

400 |

That error means your firewall **already has an SSH rule that allows access from anywhere** (`0.0.0.0/0`). You cannot add a second, more specific rule for the same port while the "allow all" rule exists.

The solution is to **edit the existing SSH rule**, not add a new one.

***

### ## How to Fix It

1.  **Go to the Networking Tab:** On your Lightsail console, navigate to the **"Networking"** tab for your `lexshift-new` instance (`3.150.250.227`).

2.  **Find the Existing SSH Rule:** You will see a rule already listed for **SSH (Port 22)**. The source for this rule will be `0.0.0.0/0`.
    

3.  **Edit the Rule:** Click the **pencil icon** (edit) next to that specific SSH rule.

4.  **Change the Source:** A dialog box will appear. Change the source from "Anywhere (0.0.0.0/0)" to **"Custom"**.

5.  **Enter the IP Address:** In the text box that appears, enter the static IP of your **old server**: `18.221.253.166`.

6.  **Save the Rule:** Click the save icon to apply the change.

This modifies the existing rule from "allow anyone" to "allow only your old server." This is also much more secure.

Once you save the change, run your `scp` command again. It should now connect without timing out.


| User Prompt: |
|-------------|
| Changed the firewall rule and got this:

  

PS C:\\Users\\Jeffrey> scp -i "D:\\Users\\Jeffrey\\OneDrive - lexshift\\documents\\businesses\\lexshift\\Lightsail SSH Keys\\LightsailDefaultKey-us-east-2.pem" -r bitnami@18.221.253.166:/opt/bitnami/wordpress/wp-content/uploads bitnami@3.150.250.227:/opt/bitnami/wordpress/wp-content/

ssh: connect to host 3.150.250.227 port 22: Connection timed out

C:\\WINDOWS\\System32\\OpenSSH\\scp.exe: Connection closed

PS C:\\Users\\Jeffrey>

  

Please advise. |

This timeout is happening because when you restricted the firewall, you locked out your own computer.

The `scp` command runs from your local PC, which has its own unique IP address. Your new firewall rule tells the `lexshift-new` server to **only** accept SSH connections from the `lexshift-old` server's IP (`18.221.253.166`), so it is now correctly rejecting the connection attempt from your PC.

To fix this, we need to add a second rule that also allows your computer to connect.

***

### ## How to Fix It

First, you need to find your computer's current public IP address.

* **Action:** Open a new browser tab and search Google for **"what is my ip"**. Your public IP address will be displayed at the top of the search results.
    

***

Now, let's add that IP to the firewall.

1.  **Navigate to the Lightsail Console:** Go to the **"Networking"** tab for your `lexshift-new` instance (`3.150.250.227`).
2.  **Add a New Firewall Rule:** Click **"Add rule"**.
    * **Application:** Select **SSH**.
    * **Source:** Change the source to **"Custom"**.
    * **IP Address:** Enter the public IP address you just found for your computer.
3.  **Save the Rule:** Click the save icon to create the rule.

You will now have two SSH rules in your firewall: one that allows the old server to connect and one that allows your local computer to connect. This is both secure and functional.

After waiting a moment for the rule to apply, run your `scp` command again. It should now connect successfully.


| User Prompt: |
|-------------|
| Added the second firewall rule and same result. Please advise. |

You've configured the firewalls perfectly. The fact that the "Connection timed out" error still occurs points to a subtle issue with how `scp` handles authentication when transferring files between two remote servers.

The command you're running tells your old server to initiate a connection to the new one, but the old server doesn't have the necessary SSH key to complete that connection. Your local PC has the key, but it can't pass it along for that second step.

The most reliable solution is to change our strategy slightly and use your local computer as a middle-step. We will download the files from the old server to your PC, and then upload them from your PC to the new server. This bypasses the server-to-server authentication problem completely.

-----

### \#\# New Plan: Download then Upload

We will run two separate `scp` commands from your PowerShell terminal.

#### 1\. Download Files from Old Server to Your PC

This command will copy the entire `uploads` folder from `lexshift-old` into the directory you are currently in on your computer.

1.  **Make sure you are in a suitable directory** in PowerShell (like `D:\Users\Jeffrey\Downloads` or a new folder).
2.  **Run the download command:**
    ```powershell
    scp -i "D:\Users\Jeffrey\OneDrive - lexshift\documents\businesses\lexshift\Lightsail SSH Keys\LightsailDefaultKey-us-east-2.pem" -r bitnami@18.221.253.166:/opt/bitnami/wordpress/wp-content/uploads .
    ```
    *Notice the `.` at the very end.* This tells `scp` to copy the folder into your current local directory. Wait for this process to complete.

-----

#### 2\. Upload Files from Your PC to the New Server

Once the download is finished, you will have an `uploads` folder on your computer. Now we'll push that folder up to the `lexshift-new` server.

  * **Run the upload command:**
    ```powershell
    scp -i "D:\Users\Jeffrey\OneDrive - lexshift\documents\businesses\lexshift\Lightsail SSH Keys\LightsailDefaultKey-us-east-2.pem" -r uploads bitnami@3.150.250.227:/opt/bitnami/wordpress/wp-content/
    ```

This two-step method is guaranteed to work because each command is a simple, direct connection from your computer to a server, which your firewall rules and SSH key are already set up to handle. After the upload is complete, you can proceed with registering the files in the media library as we discussed previously.


| User Prompt: |
|-------------|
| Please advise:

  

PS C:\\Users\\Jeffrey> cd downloads

PS C:\\Users\\Jeffrey\\downloads> scp -i "D:\\Users\\Jeffrey\\OneDrive - lexshift\\documents\\businesses\\lexshift\\Lightsail SSH Keys\\LightsailDefaultKey-us-east-2.pem" -r bitnami@18.221.253.166:/opt/bitnami/wordpress/wp-content/uploads .

The authenticity of host '18.221.253.166 (18.221.253.166)' can't be established.

ED25519 key fingerprint is SHA256:gXB4Gmb7nE2J7zZvCXlbGMWKODBH9wDx1In4K+PZT3c.

This key is not known by any other names.

Are you sure you want to continue connecting (yes/no/\[fingerprint\])?

Please type 'yes', 'no' or the fingerprint:

Warning: Permanently added '18.221.253.166' (ED25519) to the list of known hosts.

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

Permissions for 'D:/Users/Jeffrey/OneDrive - lexshift/documents/businesses/lexshift/Lightsail SSH Keys/LightsailDefaultKey-us-east-2.pem' are too open.

It is required that your private key files are NOT accessible by others.

This private key will be ignored.

Load key "D:/Users/Jeffrey/OneDrive - lexshift/documents/businesses/lexshift/Lightsail SSH Keys/LightsailDefaultKey-us-east-2.pem": bad permissions

bitnami@18.221.253.166: Permission denied (publickey).

C:\\WINDOWS\\System32\\OpenSSH\\scp.exe: Connection closed

PS C:\\Users\\Jeffrey\\downloads> |

This is a security error. Your computer is correctly refusing to use the `.pem` key file because its file permissions are too open, meaning other user accounts on your PC could potentially access it. We just need to lock down the file so only you can read it.

Since you're on Windows, you'll do this through the file's properties.

***

### ## How to Fix "Bad Permissions" on Windows

1.  **Navigate to Your Key File:** Open File Explorer and go to the folder containing your `.pem` key:
    `D:\Users\Jeffrey\OneDrive - lexshift\documents\businesses\lexshift\Lightsail SSH Keys\`

2.  **Open Security Properties:**
    * Right-click on the `LightsailDefaultKey-us-east-2.pem` file.
    * Select **Properties**.
    * Go to the **Security** tab and click the **Advanced** button.
    

3.  **Disable Inheritance:**
    * At the bottom of the "Advanced Security Settings" window, click the **Disable inheritance** button.
    * A box will pop up. Choose the first option: **"Convert inherited permissions into explicit permissions on this object."**

4.  **Remove Other Users:**
    * You will see a list of users and groups (like "SYSTEM," "Administrators," etc.).
    * One by one, click on every entry **EXCEPT** your own user account (`Jeffrey`) and click the **Remove** button.
    * When you're done, the **only entry left should be your own user account**.

5.  **Apply and Close:**
    * Click **Apply**, then **OK**.
    * Click **OK** again to close the properties window.

The file is now properly secured. Go back to your PowerShell window and run the same `scp` download command again. It will now work correctly.


| User Prompt: |
|-------------|
| Done and now this:

  

PS C:\\Users\\Jeffrey\\downloads> scp -i "D:\\Users\\Jeffrey\\OneDrive - lexshift\\documents\\businesses\\lexshift\\Lightsail SSH Keys\\LightsailDefaultKey-us-east-2.pem" -r bitnami@18.221.253.166:/opt/bitnami/wordpress/wp-content/uploads .

Load key "D:/Users/Jeffrey/OneDrive - lexshift/documents/businesses/lexshift/Lightsail SSH Keys/LightsailDefaultKey-us-east-2.pem": invalid format

bitnami@18.221.253.166: Permission denied (publickey).

C:\\WINDOWS\\System32\\OpenSSH\\scp.exe: Connection closed

PS C:\\Users\\Jeffrey\\downloads> |

This "invalid format" error means that the OpenSSH client used by PowerShell cannot parse the `.pem` key file. This is a common issue when a key has been handled by different systems (like PuTTY and Windows) which can sometimes cause subtle formatting problems.

The most reliable solution is to use **PuTTYgen** (which you should have alongside PuTTY) to re-export the key in a clean, universally compatible OpenSSH format.

-----

### \#\# The Solution: Re-export the Key with PuTTYgen

This process will create a new, clean key file that we'll use for the command.

1.  **Open PuTTYgen:** Launch the PuTTYgen application on your computer.

2.  **Load Your Existing Key:**

      * Click the **Load** button.
      * In the file open window, change the file type dropdown from "PuTTY Private Key Files (\*.ppk)" to **"All Files (\*.\*)"**.
      * Select your `LightsailDefaultKey-us-east-2.pem` file and click **Open**.

3.  **Export the Key:**

      * PuTTYgen will now show the key details. Do **NOT** click "Save private key".
      * Go to the menu at the top, click **Conversions**, and then select **"Export OpenSSH key"**.

4.  **Save the New Key File:**

      * A "Save As" window will appear. Save the new key in the **same folder** as your original.
      * Name it something distinct, for example: `LightsailDefaultKey-OpenSSH.pem`.

-----

### \#\# Final Steps: Permissions and Execution

1.  **Fix Permissions on the NEW File:** The new file you just created will have the same "permissions too open" problem. You must repeat the exact same steps you did before to restrict its security permissions in Windows File Explorer, leaving only your user account with access.

2.  **Run the Corrected Command:** Go back to PowerShell and run the `scp` command, but this time, point it to your **newly created key file**.

    ```powershell
    scp -i "D:\Users\Jeffrey\OneDrive - lexshift\documents\businesses\lexshift\Lightsail SSH Keys\LightsailDefaultKey-OpenSSH.pem" -r bitami@18.221.253.166:/opt/bitnami/wordpress/wp-content/uploads .
    ```

This will resolve the "invalid format" error and the file transfer should now begin.


| User Prompt: |
|-------------|
| Excellent. That worked. Should I retry the direct copy? |

No, you should **not** retry the direct copy.

The reason the download worked is that we solved the authentication issue **between your PC and the server**. The original problem of the old server authenticating with the new server still exists.

Stick with the two-step plan. It's guaranteed to work, and you're already halfway there.

-----

### \#\# Next Step: Upload to the New Server

Now, run the second `scp` command to upload the `uploads` folder from your computer to the new `lexshift-new` server.

```powershell
scp -i "D:\Users\Jeffrey\OneDrive - lexshift\documents\businesses\lexshift\Lightsail SSH Keys\LightsailDefaultKey-OpenSSH.pem" -r uploads bitnami@3.150.250.227:/opt/bitnami/wordpress/wp-content/
```

Once this is finished, you can proceed with the **Media Sync** plugin on your new site to get everything registered.


| User Prompt: |
|-------------|
| There appear to be a lot of folders in the \`uploads\` directory beyond the media files that I want to migrate. Should I be selective in what I upload to the new site? |




| User Prompt: |
|-------------|
| There appear to be a lot of folders in the \`uploads\` directory beyond the media files that I want to migrate. Should I be selective in what I upload to the new site?

  

\`\`\`ps

    Directory: C:\\Users\\Jeffrey\\downloads\\uploads

  

Mode                 LastWriteTime         Length Name

\----                 -------------         ------ ----

lar--          10/14/2025  5:41 PM                2022

lar--          10/14/2025  5:41 PM                2023

lar--          10/14/2025  5:41 PM                2024

lar--          10/14/2025  5:41 PM                2025

lar--          10/14/2025  5:41 PM                backup

lar--          10/14/2025  5:41 PM                eb-patterns

lar--          10/14/2025  5:41 PM                elementor

lar--          10/14/2025  5:41 PM                essential-addons-elementor

lar--          10/14/2025  5:41 PM                happyaddons

lar--          10/14/2025  5:41 PM                jetbackup

lar--          10/14/2025  5:41 PM                sg\_symlinks

lar--          10/14/2025  5:41 PM                smush

lar--          10/14/2025  5:41 PM                themeisle-gutenberg

lar--          10/14/2025  5:41 PM                unlimited\_elements\_cache

lar--          10/14/2025  5:41 PM                wp-migrate-db

lar--          10/14/2025  5:38 PM                wp-staging

lar--          10/14/2025  5:41 PM                wp-staging-sites

lar--          10/14/2025  5:41 PM                wpforms

lar--          10/14/2025  5:41 PM                wpr-addons

lar--          10/14/2025  5:41 PM                wpsynchro-665dd7520937b

la---          10/14/2025  5:39 PM            469 smush-webp-test.png

  

PS C:\\Users\\Jeffrey\\downloads\\uploads>

\`\`\` |

Yes, you should absolutely be selective. Uploading only the necessary folders will make the transfer much faster and prevent potential conflicts from old plugin data on your new, clean site.

Looking at your list, we can separate the folders into two categories: the essential media files and the non-essential plugin/backup data.

-----

### ✅ What to Keep (The Must-Haves)

The only folders you need to migrate are the ones containing your actual media assets. WordPress organizes these into year- and month-based directories.

  * **`2022`**
  * **`2023`**
  * **`2024`**
  * **`2025`**

These folders contain all the images, PDFs, and other files you've uploaded to the media library.

-----

### 🧹 What to Exclude (The Leave Behinds)

You can safely ignore all the other folders. They are generated by plugins and are not part of your core media library. Bringing them over is unnecessary and could cause issues.

  * **Plugin Data:** Folders like `elementor`, `wpforms`, `smush`, `happyaddons`, etc., are created by those plugins for their own caching, styles, and temporary files. The fresh versions of these plugins on your new site will create their own clean folders.
  * **Backup & Staging Data:** Folders like `backup`, `wp-staging`, and `wpsynchro` are from your old site's maintenance tools. They are completely irrelevant to the new site and would just be dead weight.

-----

### 🛠️ The Action Plan: Uploading Only the Right Folders

We will adjust the `scp` command to upload only the four year-based folders. This is much more efficient.

1.  **Ensure you are in the right local directory:** Your PowerShell prompt should be `PS C:\Users\Jeffrey\downloads\uploads>`.

2.  **Run the selective upload command:** This command will copy only the specified year folders into the `uploads` directory on the new server.

    ```powershell
    scp -i "D:\Users\Jeffrey\OneDrive - lexshift\documents\businesses\lexshift\Lightsail SSH Keys\LightsailDefaultKey-OpenSSH.pem" -r 2022 2023 2024 2025 bitnami@3.150.250.227:/opt/bitnami/wordpress/wp-content/uploads/
    ```

After this command completes, your core media files will be in the correct location on the new server. You can then proceed with the **Media Sync** plugin step to register them in the WordPress library.


| User Prompt: |
|-------------|
| Please advise. Note the name of the key file is correct:

  

\`\`\`ps

PS C:\\Users\\Jeffrey\\downloads\\uploads> scp -i "D:\\Users\\Jeffrey\\OneDrive - lexshift\\documents\\businesses\\lexshift\\Lightsail SSH Keys\\LightsailDefaultKey-us-east-2\_OpenSSH.pem" -r 2022 2023 2024 2025 bitnami@3.150.250.227:/opt/bitnami/wordpress/wp-content/uploads/

ssh: connect to host 3.150.250.227 port 22: Connection timed out

C:\\WINDOWS\\System32\\OpenSSH\\scp.exe: Connection closed

PS C:\\Users\\Jeffrey\\downloads\\uploads>

\`\`\` |

This is the same "Connection timed out" error we encountered before. It confirms that the firewall on your **new server (`3.150.250.227`)** is actively blocking the connection from your computer.

The most likely reason for this is that your computer's public IP address has changed since you added it to the firewall rule. This is very common for home and office internet connections.

***

### ## How to Fix It

We just need to update the firewall rule with your current IP address.

1.  **Find Your Current Public IP:**
    * Open a new browser tab and search Google for **"what is my ip"**.
    * Your current public IP address will be displayed at the top. Note it down.
    

2.  **Update the Lightsail Firewall Rule:**
    * Go to the AWS Lightsail console and navigate to the **Networking** tab for your `lexshift-new` instance (`3.150.250.227`).
    * Find the SSH rule you created for your computer's IP address.
    * Click the **pencil icon (edit)** next to that rule.
    * **Replace the old IP address** with the new one you just found.
    * Click the save icon to apply the change.

3.  **Re-run the Command:**
    * Wait about a minute for the firewall change to take effect.
    * Go back to your PowerShell terminal and run the exact same `scp` upload command again.

The connection should now succeed, and the file transfer will begin.


| User Prompt: |
|-------------|
| My IP address has not changed and is still present in the firewall rule. Please advise. |

This is a frustrating situation, especially since you've set up the firewall rules correctly. A "Connection timed out" error when the rules are right almost always means something outside of the Lightsail firewall is blocking the connection.

Let's use a different tool to both transfer the files and diagnose the problem: **FileZilla**. It provides more detailed connection logs and is often more resilient to network issues than the command-line `scp` client.

***

### ## Using FileZilla to Upload and Troubleshoot

This will replace the `scp` upload command.

1.  **Open FileZilla:** Launch the application on your computer.

2.  **Open the Site Manager:**
    * Click on **File** -> **Site Manager**.
    * Click **New Site** and name it "LexShift New".

3.  **Configure the Connection:**
    * **Protocol:** Select `SFTP - SSH File Transfer Protocol`.
    * **Host:** Enter the IP for your new server: `3.150.250.227`.
    * **Logon Type:** Change to **"Key file"**.
    * **User:** Enter `bitnami`.
    * **Key file:** Browse and select your **newly exported OpenSSH key file**: `D:\...\LightsailDefaultKey-OpenSSH.pem`.
    

4.  **Connect and Observe:**
    * Click the **Connect** button.
    * Pay close attention to the status messages in the top log window.

***

### ## Interpreting the Results

* **If FileZilla connects successfully:** The issue is with your local `scp` client or its environment in PowerShell. In this case, simply use FileZilla to drag the `2022`, `2023`, `2024`, and `2025` folders from your local machine (left pane) to the `/opt/bitnami/wordpress/wp-content/uploads/` directory on the server (right pane).

* **If FileZilla also times out:** This confirms the block is external. The most likely culprit is a **firewall on your local computer** (like Windows Defender Firewall) or a network-level block from your Internet Service Provider. You can temporarily disable your Windows Firewall and try connecting with FileZilla again to confirm if that's the source.

Using FileZilla is the most efficient next step to bypass this roadblock and get your files uploaded.


| User Prompt: |
|-------------|
| It appears that all files that were transferring into the \*\*existing\*\* 2025 folder in the new instance errored for permissions reasons. Files that were transferred to \*\*new\*\* folders (2022, 2023, 2024) were fine. Please troubleshoot. |

This is a classic file ownership issue. Your diagnosis is spot on.

-----

### \#\# The Cause: File Ownership Mismatch 🧐

Here's exactly what happened:

  * **Existing `2025` Folder:** This folder was created by WordPress on your new server when you uploaded your first media file this year. The web server (Apache) runs as the **`daemon`** user, so it created the folder and assigned ownership to `daemon`.
  * **New `2022`, `2023`, `2024` Folders:** When you started your upload, your SFTP client (FileZilla) connected as the **`bitnami`** user. Since these folders didn't exist, FileZilla created them, making `bitnami` the owner.
  * **The Conflict:** When you tried to upload files into the `2025` folder, you were acting as the `bitnami` user trying to write into a folder owned by the `daemon` user. The server's permissions correctly blocked this action.

-----

### \#\# The Solution: Change Folder Ownership

The fix is simple. We just need to tell the server that the `bitnami` user should be the owner of the `2025` folder. You'll need to do this via your PuTTY/SSH connection to the **new server (`3.150.250.227`)**.

1.  **Navigate to the Uploads Directory:**

    ```bash
    cd /opt/bitnami/wordpress/wp-content/uploads/
    ```

2.  **Verify the Ownership (Optional but Recommended):**
    Run the command `ls -l`. You'll see an output where the `2025` folder is owned by `daemon daemon`, while the others are owned by `bitnami daemon`.

3.  **Change Ownership of the `2025` Folder:**
    This command recursively changes the owner and group for the `2025` folder and everything inside it.

    ```bash
    sudo chown -R bitnami:daemon 2025
    ```

4.  **Retry the Upload:**
    Go back to FileZilla and re-queue the files that failed to transfer into the `2025` folder. They should now upload without any permission errors. 👍

Once all the files are successfully transferred, you can proceed with the **Media Sync** plugin to register them in the WordPress database.


| User Prompt: |
|-------------|
| All files have transferred. Let's proceed to the sync step. |

Excellent. The file transfer is the most difficult part, and it's now complete.

The final step is to tell your new WordPress site about these files. We'll use the **Media Sync** plugin to scan the `uploads` folder and import everything into the Media Library database.

Here are the step-by-step instructions to follow on your **new site**.

***

### 1. Log In to Your New Site's Dashboard

* Navigate to your new site's admin area: `https://lexshift.com/admin`
* Log in with your administrator credentials.

### 2. Install and Activate the Media Sync Plugin

1.  In the left-hand menu, navigate to **Plugins** -> **Add New**.
2.  In the search bar, type **"Media Sync"**.
3.  Find the plugin by Ewww.io and click **"Install Now"**.
4.  After installation, click **"Activate"**.

### 3. Run the Sync Process

1.  In the left-hand menu, a new option will be available. Go to **Media** -> **Media Sync**.
2.  The plugin will automatically scan your `uploads` directory and show you a list of all the files that are present on the server but not yet registered in the Media Library.
3.  Review the list to ensure it looks correct.
4.  Click the **"Sync"** or **"Scan All"** button to begin the import process.

This may take a few minutes depending on the number of files. The plugin will provide a progress bar.

### 4. Verify and Clean Up

1.  **Check the Media Library:** Once the sync is complete, navigate to **Media** -> **Library**. All the images and documents from your old site should now be visible and usable.

2.  **Test a Link:** To be absolutely certain, test one of your key image URLs. For example, navigate directly to:
    `https://lexshift.com/wp-content/uploads/2023/10/horiz_dark_logo.png`
    The image should load correctly.

3.  **Deactivate and Delete the Plugin:** Media Sync is a one-time utility. Once you have confirmed that all your files are present in the library, it is best practice to deactivate and delete the plugin to keep your site clean.
    * Go to **Plugins** -> **Installed Plugins**.
    * Find Media Sync, click **"Deactivate"**, and then click **"Delete"**.

Your media migration is now complete.


| User Prompt: |
|-------------|
| What should I select for date/time to set for newly imported files? Options are default, file time, and smart file time. |

You should select **Smart File Time**.

***

### ## Why This Is the Best Option

* **Smart File Time:** This setting is the most intelligent. It looks at the folder structure where the file is located (e.g., `/wp-content/uploads/2023/10/`) and uses that information to set the import date. This will preserve the original upload dates, keeping your media library correctly organized by when the files were originally added. 🗓️

* **Default:** This would assign today's date and time to every single file, which would incorrectly show that all your media was uploaded at once.

* **File Time:** This would use the "last modified" timestamp from the file system, which would be the time you just finished uploading them. This is also incorrect.


| User Prompt: |
|-------------|
| Excellent. Now, I'd like to review the plugins on the old site and selectively install them on the new site, as appropriate. How do I get a list of the installed plugins on the old site for you to review? |

You can get a clean, complete list of your plugins using a single command through your PuTTY connection. This is the fastest and easiest way to get the information in a format you can copy and paste right back here.

-----

### \#\# Get Plugin List via Command Line (Recommended)

1.  **Connect to Your OLD Server:** Open PuTTY and start an SSH session with your `lexshift-old` instance at **`18.221.253.166`**. Log in as the `bitnami` user.

2.  **Navigate to the WordPress Directory:** Once you're logged in, go to the correct folder with this command:

    ```bash
    cd /opt/bitnami/wordpress
    ```

3.  **Run the WP-CLI Command:** Execute the following command to generate a formatted table of all installed plugins.

    ```bash
    wp plugin list
    ```

4.  **Copy and Paste the Output:** The terminal will display a table with the plugin name, status (active/inactive), update availability, and version. Simply highlight the entire table in your PuTTY window, and it will be copied to your clipboard. You can then paste it directly into our chat. 👍

-----

### \#\# Alternative: WordPress Dashboard

If you prefer, you can also get the list from the WordPress admin panel.

1.  **Log in** to your old site's dashboard at `http://18.221.253.166/admin`.
2.  Navigate to **Plugins** -\> **Installed Plugins**.
3.  You can review the **Active** and **Inactive** lists and manually type out the ones you'd like me to review.

I recommend the command-line method as it's quicker and provides a more comprehensive overview.


| User Prompt: |
|-------------|
| Please see the following and advise:

  

Here's the plugin list from the old site:

  

\`\`\`bash

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ sudo wp plugin list

+-------------------------------------+----------+-----------+-----------+

| name                                | status   | update    | version   |

+-------------------------------------+----------+-----------+-----------+

| widgetkit-for-elementor             | inactive | available | 2.5.6     |

| all-in-one-wp-migration             | inactive | none      | 7.100     |

| all-in-one-seo-pack                 | inactive | none      | 4.8.8     |

| better-search-replace               | active   | none      | 1.4.10    |

| cron-logger                         | active   | none      | 1.3.0     |

| elementor                           | active   | none      | 3.32.4    |

| elementor-pro                       | active   | none      | 3.32.2    |

| essential-addons-for-elementor-lite | inactive | none      | 6.3.3     |

| essential-addons-elementor          | inactive | none      | 5.8.1     |

| filebird                            | active   | none      | 6.4.9     |

| fluentform                          | active   | none      | 6.1.4     |

| happy-elementor-addons              | active   | none      | 3.20.1    |

| ht-slider-for-elementor             | active   | available | 1.7.1     |

| malcare-security                    | active   | none      | 6.02      |

| resmushit-image-optimizer           | inactive | none      | 1.0.4     |

| royal-elementor-addons              | active   | none      | 1.7.1035  |

| wp-smushit                          | active   | none      | 3.22.1    |

| svg-support                         | active   | none      | 2.5.14    |

| header-footer-elementor             | active   | none      | 2.5.2     |

| updraftplus                         | active   | none      | 2.25.8.26 |

| w3-total-cache                      | active   | none      | 2.8.13    |

| wordfence                           | active   | none      | 8.1.0     |

| wp-crontrol                         | active   | none      | 1.19.2    |

| wpforms-lite                        | active   | available | 1.9.8.1   |

| wp-mail-smtp                        | active   | none      | 4.6.0     |

| wp-staging-pro                      | active   | none      | 6.3.2     |

| wp-staging                          | active   | none      | 4.3.2     |

| wpsynchro                           | active   | none      | 1.13.0    |

| wpvivid-backuprestore               | active   | none      | 0.9.119   |

| local-by-flywheel-live-link-helper  | must-use | none      | 2.0       |

| wp-staging-optimizer                | must-use | none      | 1.6.0     |

| wpsynchro-mu-plugin-compat          | must-use | none      | 1.0.5     |

| advanced-cache.php                  | dropin   | none      |           |

+-------------------------------------+----------+-----------+-----------+

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$

\`\`\`

  

And here's the list from the new site:

  

\`\`\`bash

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress/wp-content/uploads$ sudo wp plugin list

+-----------------------------------+----------+-----------+---------+----------------+-------------+

| name                              | status   | update    | version | update\_version | auto\_update |

+-----------------------------------+----------+-----------+---------+----------------+-------------+

| akismet                           | inactive | none      | 5.5     |                | on          |

| all-in-one-wp-migration           | inactive | none      | 7.100   |                | on          |

| better-search-replace             | active   | none      | 1.4.10  |                | on          |

| creative-mail-by-constant-contact | active   | none      | 1.6.9   |                | on          |

| duplicate-page                    | active   | none      | 4.5.5   |                | on          |

| famethemes-demo-importer          | inactive | none      | 1.1.11  |                | on          |

| olympus-google-fonts              | active   | none      | 3.9.8   |                | on          |

| google-analytics-for-wordpress    | active   | none      | 9.8.0   |                | on          |

| hello                             | inactive | none      | 1.7.2   |                | on          |

| jetpack                           | active   | none      | 15.1.1  |                | on          |

| media-sync                        | active   | none      | 1.4.8   |                | off         |

| onepress-plus                     | active   | available | 2.3.8   | 2.3.10         | on          |

| optinmonster                      | active   | none      | 2.16.21 |                | on          |

| bluehost-wordpress-plugin         | inactive | none      | 4.7.0   |                | on          |

| updraftplus                       | active   | none      | 1.25.8  |                | off         |

| wpforms-lite                      | active   | none      | 1.9.8.2 |                | on          |

| wp-mail-smtp                      | active   | none      | 4.6.0   |                | off         |

| wordpress-seo                     | inactive | none      | 26.1.1  |                | on          |

| sso                               | must-use |           | 0.5     |                | off         |

+-----------------------------------+----------+-----------+---------+----------------+-------------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress/wp-content/uploads$

\`\`\` |

Of course. Thank you for providing both lists. This is the perfect way to create a lean, efficient, and modern setup on your new site.

My analysis is broken down into four groups: essential plugins to install, recommended utilities, a decision you need to make, and plugins you can safely leave behind.

***

### ✅ Group 1: Essential Plugins to Install

These are the core components of your site's design and functionality. You should install **fresh copies** of these from the WordPress repository or your premium plugin accounts.

* **`elementor` & `elementor-pro`**: The core page builder for your site. This is non-negotiable.
* **`happy-elementor-addons`**: An active add-on that is likely providing specific widgets or styles used in your design.
* **`ht-slider-for-elementor`**: Provides slider functionality used on your site.
* **`royal-elementor-addons`**: Another active add-on providing design elements.
* **`header-footer-elementor`**: This is critical for building your site's global header and footer.
* **`svg-support`**: Enables the use of SVG logos and icons, which is important for modern design.

***

### 🛠️ Group 2: Recommended Utility Plugins to Install

These plugins handle important background tasks like security, performance, and site management.

* **`w3-total-cache`**: Your chosen performance and caching plugin. It's essential for site speed.
* **`malcare-security`**: A good security and malware scanning plugin. Since your new site doesn't have a dedicated security plugin yet (Jetpack has some features, but this is more focused), this is a solid choice to bring over.
* **`wp-smushit`**: Your image optimization plugin. This is key for keeping page load times fast.
* **`filebird`**: A "quality of life" plugin that helps organize your media library into folders. If you liked this functionality on the old site, bring it over.
* **`wp-crontrol`**: A very useful tool for viewing and managing WordPress cron jobs (scheduled tasks). It's great for troubleshooting.

***

### 🤔 Group 3: Your Decision Needed (Forms)

Your old site and new site use different form builders.

* **Old Site:** You were using **`fluentform`**.
* **New Site:** You currently have **`wpforms-lite`**.

You need to decide which one you want to use going forward.
* **Option A (Easiest):** Rebuild your old forms using **WPForms Lite** on the new site. If you only have a simple contact form, this will take just a few minutes.
* **Option B (More Complex):** Install **Fluentform** on the new site and use its export/import tools to migrate your existing forms. Choose this if you have complex forms that would be difficult to rebuild.

I recommend keeping only **one** form plugin active to keep the site lean.

***

### 🧹 Group 4: Plugins to Leave Behind

You can safely ignore all of these. They are either inactive, redundant, or were for development/migration purposes.

* **Inactive Plugins:** `widgetkit-for-elementor`, `all-in-one-seo-pack`, `essential-addons-*`, `resmushit-image-optimizer`. If they were inactive, you don't need them.
* **Migration/Staging/Backup Tools:** `all-in-one-wp-migration`, `wp-staging-pro`, `wp-staging`, `wpsynchro`, `wpvivid-backuprestore`. Their job is done, or they are redundant with `UpdraftPlus` which is already on the new site.
* **Redundant Security:** `wordfence`. You already have `malcare-security` active. It's best practice to run only one primary security plugin.
* **Plugins Already on New Site:** `better-search-replace`, `updraftplus`, `wpforms-lite`, `wp-mail-smtp`. No action needed here.
* **`must-use` & `dropin` plugins:** Do not manually copy these. They are typically installed by your hosting environment or other plugins and will be created as needed on the new site.

***

### Action Plan

1.  **Install the "Essential" plugins** from Group 1.
2.  **Install the "Recommended" plugins** from Group 2.
3.  **Decide** which form plugin you will use from Group 3 and take the appropriate action.
4.  **Configure** your key utilities: set up a backup schedule in `UpdraftPlus`, run a scan with `malcare-security`, and configure `W3 Total Cache`.
5.  **Deactivate and delete `Media Sync`**; its job is done.


| User Prompt: |
|-------------|
| Following are the two plugin lists. Please advise next steps:

  

\`\`\`bash

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ sudo wp plugin list

+------------------------------------+----------+-----------+-----------+

| name                               | status   | update    | version   |

+------------------------------------+----------+-----------+-----------+

| better-search-replace              | active   | none      | 1.4.10    |

| cron-logger                        | active   | none      | 1.3.0     |

| elementor                          | active   | none      | 3.32.4    |

| elementor-pro                      | active   | none      | 3.32.2    |

| filebird                           | active   | none      | 6.4.9     |

| fluentform                         | active   | none      | 6.1.4     |

| happy-elementor-addons             | active   | none      | 3.20.1    |

| ht-slider-for-elementor            | active   | available | 1.7.1     |

| malcare-security                   | active   | none      | 6.02      |

| royal-elementor-addons             | active   | none      | 1.7.1035  |

| wp-smushit                         | active   | none      | 3.22.1    |

| svg-support                        | active   | none      | 2.5.14    |

| header-footer-elementor            | active   | none      | 2.5.2     |

| updraftplus                        | active   | none      | 2.25.8.26 |

| w3-total-cache                     | active   | none      | 2.8.13    |

| wordfence                          | active   | none      | 8.1.0     |

| wp-crontrol                        | active   | none      | 1.19.2    |

| wpforms-lite                       | active   | available | 1.9.8.1   |

| wp-mail-smtp                       | active   | none      | 4.6.0     |

| wp-staging-pro                     | active   | none      | 6.3.2     |

| wp-staging                         | active   | none      | 4.3.2     |

| wpsynchro                          | active   | none      | 1.13.0    |

| wpvivid-backuprestore              | active   | none      | 0.9.119   |

| local-by-flywheel-live-link-helper | must-use | none      | 2.0       |

| wp-staging-optimizer               | must-use | none      | 1.6.0     |

| wpsynchro-mu-plugin-compat         | must-use | none      | 1.0.5     |

| advanced-cache.php                 | dropin   | none      |           |

+------------------------------------+----------+-----------+-----------+

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$

\`\`\`

  

\`\`\`bash

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress/wp-content/uploads$ sudo wp plugin list

+-----------------------------------+----------+-----------+----------+----------------+-------------+

| name                              | status   | update    | version  | update\_version | auto\_update |

+-----------------------------------+----------+-----------+----------+----------------+-------------+

| akismet                           | inactive | none      | 5.5      |                | on          |

| all-in-one-wp-migration           | inactive | none      | 7.100    |                | on          |

| better-search-replace             | active   | none      | 1.4.10   |                | on          |

| creative-mail-by-constant-contact | active   | none      | 1.6.9    |                | on          |

| duplicate-page                    | active   | none      | 4.5.5    |                | on          |

| elementor                         | active   | none      | 3.32.4   |                | off         |

| famethemes-demo-importer          | inactive | none      | 1.1.11   |                | on          |

| filebird                          | inactive | none      | 6.4.9    |                | off         |

| olympus-google-fonts              | active   | none      | 3.9.8    |                | on          |

| google-analytics-for-wordpress    | active   | none      | 9.8.0    |                | on          |

| happy-elementor-addons            | active   | none      | 3.20.1   |                | off         |

| hello                             | inactive | none      | 1.7.2    |                | on          |

| ht-slider-for-elementor           | inactive | none      | 1.7.2    |                | off         |

| jetpack                           | active   | none      | 15.1.1   |                | on          |

| malcare-security                  | inactive | none      | 6.02     |                | off         |

| media-sync                        | active   | none      | 1.4.8    |                | off         |

| onepress-plus                     | active   | available | 2.3.8    | 2.3.10         | on          |

| optinmonster                      | active   | none      | 2.16.21  |                | on          |

| royal-elementor-addons            | inactive | none      | 1.7.1035 |                | off         |

| wp-smushit                        | inactive | none      | 3.22.1   |                | off         |

| svg-support                       | active   | none      | 2.5.14   |                | off         |

| bluehost-wordpress-plugin         | inactive | none      | 4.7.0    |                | on          |

| header-footer-elementor           | inactive | none      | 2.5.2    |                | off         |

| updraftplus                       | active   | none      | 1.25.8   |                | off         |

| w3-total-cache                    | inactive | none      | 2.8.13   |                | off         |

| wp-crontrol                       | inactive | none      | 1.19.2   |                | off         |

| wpforms-lite                      | active   | none      | 1.9.8.2  |                | on          |

| wp-mail-smtp                      | active   | none      | 4.6.0    |                | off         |

| wordpress-seo                     | inactive | none      | 26.1.1   |                | on          |

| sso                               | must-use |           | 0.5      |                | off         |

+-----------------------------------+----------+-----------+----------+----------------+-------------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress/wp-content/uploads$

\`\`\` |

Excellent. You now have everything we need to create a final, optimized plugin setup for your new site. I've analyzed both lists and created a clear, step-by-step action plan.

The goal is to match the functionality of the old site, remove unnecessary clutter from the new site, and get everything configured correctly.

***

### Step 1: Cleanup - Remove Unnecessary Plugins from the New Site

First, let's declutter your new site by removing plugins that are either leftovers from a theme, redundant, or no longer needed. Log in to your new site's dashboard (`https://lexshift.com/admin`) and go to **Plugins -> Installed Plugins**.

**Deactivate and Delete** the following:

* **`Media Sync`**: Its job is done.
* **`akismet`**: You are using `Malcare` for security, which often handles spam.
* **`all-in-one-wp-migration`**: A migration tool that is no longer needed.
* **`creative-mail-by-constant-contact`**: Marketing bloat you don't use.
* **`famethemes-demo-importer`**: A one-time tool for theme setup.
* **`jetpack`**: A large, multi-purpose plugin. You are using more specific, dedicated plugins for security (`Malcare`), backups (`UpdraftPlus`), and caching (`W3 Total Cache`), making Jetpack redundant.
* **`onepress-plus`**: This appears to be a plugin for a different theme, not "Hello Elementor".
* **`bluehost-wordpress-plugin`**: Hosting-specific plugin that isn't needed for Lightsail.
* **`wordpress-seo`**: (Yoast SEO). It's best to have only one SEO plugin. Since none were active on your old site, you can remove this for now to keep things clean.

***

### Step 2: Install the Missing Premium Plugin

There is one critical plugin from your old site that is missing on the new one.

1.  **Install `Elementor Pro`**: You need to manually upload and install the premium `elementor-pro` plugin. Download the latest version from your Elementor account, then on your new site go to **Plugins -> Add New -> Upload Plugin**.

***

### Step 3: Activate and Configure Key Plugins

Many of the essential plugins are already installed but are `inactive`. Go through your plugin list and **Activate** the following:

* **`filebird`**: To organize your newly migrated media.
* **`ht-slider-for-elementor`**: To restore slider functionality.
* **`malcare-security`**: Your primary security plugin.
* **`royal-elementor-addons`**: To restore all design elements.
* **`wp-smushit`**: Your image optimization plugin.
* **`header-footer-elementor`**: Critical for your site's header and footer.
* **`w3-total-cache`**: Your performance plugin.
* **`wp-crontrol`**: Your cron job utility.

***

### Step 4: Resolve the Form Plugin Conflict

Your old site used `fluentform` but `wpforms-lite` is active on the new site. You must choose one to avoid conflicts.

* **My Recommendation:** Stick with **`wpforms-lite`** since it's already active and likely simple to configure for a contact form.
* **Action:** Deactivate and delete `fluentform` from the old site to prevent future confusion. Check your new site and quickly rebuild your contact form using `wpforms-lite`.

If your forms are very complex, you can choose to install `fluentform` on the new site and deactivate/delete `wpforms-lite` instead. **Just be sure to only have one active form plugin.**

***

### Final Checklist

After completing these steps:

1.  **Configure Backups:** Go to **Settings -> UpdraftPlus Backups** and set up a regular backup schedule to a remote location (like Google Drive or Dropbox).
2.  **Configure Caching:** Go to the **Performance** tab and run through the setup guide for **W3 Total Cache**.
3.  **Run a Security Scan:** Go to the **Malcare** dashboard and initiate your first site scan.
4.  **Review the Site:** Browse the front end of `lexshift.com` to ensure all pages, headers, footers, and sliders look correct now that all Elementor add-ons are active.

Your new site is now fully migrated, cleaned up, and configured to match the essential functionality of your old one.


| User Prompt: |
|-------------|
| I am trying to delete a staging site on the old site before I deactivate WP Staging Pro, but when I give the proper delete commands, the plugin just spins and hangs on "initializing. please wait, process is starting ...." Please advise. |

This is a very common and frustrating issue with staging plugins. The "spinning" process indicates that the background task (an AJAX request) that's supposed to handle the deletion is failing or timing out. This is often due to server resource limits, PHP execution time limits, or file permission issues.

Since you are comfortable with the command line, we can bypass the broken UI and delete the staging site manually. This is the most direct and reliable method.

-----

### 🛡️ 1. Safety First: Confirm Your Backup

Before proceeding with manual deletion, please double-check that your Lightsail snapshot of the `lexshift-old` instance is recent and available. Manual file deletion is irreversible.

-----

### 🛠️ 2. The Manual Deletion Process

This process has two parts: deleting the staging site's files and deleting its database tables. You must be connected via PuTTY to your **old server (`18.221.253.166`)**.

#### Part A: Delete the Staging Site Files

1.  **Navigate to the WP Staging Directory:** WP Staging stores its sites in a specific subfolder.

    ```bash
    cd /opt/bitnami/wordpress/wp-content/uploads/wp-staging/
    ```

2.  **Identify the Staging Site Folder:** List the contents of the directory to see the folder corresponding to your staging site.

    ```bash
    ls -l
    ```

    You will see a folder with a name like `staging-site-name` or a random string of characters.

3.  **Delete the Folder:** Use the `rm` command to recursively delete the entire folder. Replace `<staging_site_folder_name>` with the actual folder name you identified.

    **⚠️ WARNING: This command is powerful and deletes files permanently. Double-check that you have typed the folder name correctly before pressing Enter.**

    ```bash
    sudo rm -rf <staging_site_folder_name>
    ```

#### Part B: Delete the Staging Site Database Tables

The files are gone, but the staging site's database tables still exist. We will use WP-CLI to find and remove them.

1.  **Navigate to the WordPress Root:**

    ```bash
    cd /opt/bitnami/wordpress
    ```

2.  **Find the Staging Table Prefix:** WP Staging creates new tables with a unique prefix (e.g., `wpstg0_`, `wpstg1_`, etc.). First, let's list the tables to identify the correct prefix.

    ```bash
    sudo wp db tables --all-tables-with-prefix | grep 'wpstg'
    ```

    This will show you all tables belonging to WP Staging clones. Note the prefix (e.g., `wpstg1_`).

3.  **Drop the Staging Tables:** Now, we'll run a single command to find all tables with that prefix and delete them. Replace `wpstg1_*` with the correct prefix followed by a wildcard `*`. The `' '` quotes are important.

    ```bash
    sudo wp db tables 'wpstg1_*' --all-tables-with-prefix | xargs sudo wp db drop --yes
    ```

      * **How this works:** `wp db tables` lists the tables. The `|` pipe sends that list to `xargs`, which then executes the `wp db drop` command for every table in the list. The `--yes` flag confirms the action without prompting.

-----

### ✅ 3. Final Step: Deactivate and Delete the Plugin

Now that you have manually removed all traces of the staging site, you can safely go back to the WordPress admin dashboard.

1.  Refresh the WP Staging plugin page.
2.  You should now be able to **Deactivate** and **Delete** the WP Staging Pro plugin without any issues.


| User Prompt: |
|-------------|
| Please advise:

  

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress/wp-content/uploads/wp-staging$ ls -l

total 40

drwxrwxr-x 2 daemon daemon  4096 Nov  9  2023 backups

drwxrwxr-x 2 daemon daemon  4096 Oct 14 23:34 cache

\-rw-r--r-- 1 daemon daemon 11833 Oct 11 13:22 clone\_options.cache.php

\-rw-r--r-- 1 daemon daemon     0 Oct 11  2023 index.html

\-rw-r--r-- 1 daemon daemon   696 Oct 11  2023 index.php

drwxrwxr-x 2 daemon daemon  4096 Oct 14 23:32 logs

\-rw-r--r-- 1 daemon daemon     6 Sep 29 18:38 maintenance

drwxr-xr-x 3 daemon daemon  4096 Sep 29 17:39 tmp

\-rw-r--r-- 1 daemon daemon   378 Sep 29 18:20 web.config

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress/wp-content/uploads/wp-staging$ |

Thank you for providing that output. It shows us exactly what's happening, and my apologies, my initial assumption was incorrect for your setup.

The output shows that your staging site is **not** located inside the `uploads` folder. This is actually a good thing. For performance and security reasons, WP Staging places the entire cloned site in a new folder at the **main WordPress root level**, right alongside your `wp-admin` and `wp-content` directories.

Let's correct our course. The process is the same, but we just need to look in the right place.

-----

### Step 1: Find and Delete the Staging Site's Root Folder

1.  **Navigate to the WordPress Root Directory:**

    ```bash
    cd /opt/bitnami/wordpress
    ```

2.  **List the Contents:** Run `ls -l` to see all the folders. You are looking for a folder that is **not** a standard WordPress folder (like `wp-admin`, `wp-content`, `wp-includes`). The folder will likely be named after your staging site (e.g., `lexshift-staging`, `test`, `dev`, etc.).

    ```bash
    ls -l
    ```

3.  **Delete the Staging Folder:** Once you identify the correct folder name, use the `rm` command to delete it.

    **⚠️ WARNING: This command is powerful and deletes files permanently. Be absolutely certain you have identified the correct folder before pressing Enter.**

    ```bash
    sudo rm -rf <your_staging_site_folder_name>
    ```

-----

### Step 2: Delete the Staging Site's Database Tables

The next steps for deleting the database tables remain the same, as the tables are all in the same database regardless of where the files are.

1.  **Find the Staging Table Prefix:** While still in the `/opt/bitami/wordpress` directory, list the tables to find the prefix for your staging site (e.g., `wpstg0_`, `wpstg1_`).

    ```bash
    sudo wp db tables --all-tables-with-prefix | grep 'wpstg'
    ```

2.  **Drop the Staging Tables:** Use the prefix you just found to delete all corresponding tables. Replace `wpstg1_*` with the correct prefix and wildcard `*`.

    ```bash
    sudo wp db tables 'wpstg1_*' --all-tables-with-prefix | xargs sudo wp db drop --yes
    ```

-----

### Step 3: Deactivate and Delete the Plugin

Once you've deleted the root folder and the database tables, the staging site will be completely gone. You can now return to your WordPress admin dashboard and safely **Deactivate** and **Delete** the WP Staging and WP Staging Pro plugins.


| User Prompt: |
|-------------|
| Please advise:

  

\`\`\`bash

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$ ls -l

total 288

\-rwxr-xr-x  1 daemon  daemon 19052 Apr  4  2024 bv\_connector\_e9a59f81ae7d0926cc1284e67959852a.php

\-rw-rw-r--  1 bitnami daemon   405 Oct 11  2023 index.php

drwxrwxr-x  2 bitnami daemon  4096 Aug 29  2023 licenses

\-rw-rw-r--  1 bitnami daemon 19903 Jun  4 00:37 license.txt

\-rw-r--r--  1 daemon  daemon    20 Oct 11  2023 local-xdebuginfo.php

\-rw-r--r--  1 daemon  daemon   439 Jul 16  2024 malcare-waf.php

\-rw-rw-r--  1 bitnami daemon  7425 Oct  1 00:57 readme.html

drwxrwxr-x  2 bitnami daemon  4096 Aug 30  2023 tmp

\-rw-rw-r--  1 daemon  daemon   325 Oct 13  2023 wordfence-waf.php

\-rw-rw-r--  1 bitnami daemon  7387 Apr 12  2024 wp-activate.php

drwxrwxr-x  9 bitnami daemon  4096 Aug 29  2023 wp-admin

\-rw-rw-r--  1 bitnami daemon   351 Oct 11  2023 wp-blog-header.php

\-rw-rw-r--  1 bitnami daemon  2323 Oct 11  2023 wp-comments-post.php

lrwxrwxrwx  1 root    root      32 Oct 11  2023 wp-config\_OLD.php -> /bitnami/wordpress/wp-config.php

\-rw-rw-rw-  1 daemon  daemon  4825 Oct 14 17:42 wp-config.php

\-rw-r--r--  1 root    root    4721 Oct 14 17:39 wp-config.php.bak

\-rw-rw-r--  1 bitnami daemon  3336 Nov 15  2024 wp-config-sample.php

lrwxrwxrwx  1 root    root      29 Oct 11  2023 wp-content -> /bitnami/wordpress/wp-content

\-rw-rw-r--  1 bitnami daemon  5617 Nov 15  2024 wp-cron.php

drwxrwxr-x 30 bitnami daemon 16384 Jun  4 00:37 wp-includes

\-rw-rw-r--  1 bitnami daemon  2502 Oct 11  2023 wp-links-opml.php

\-rw-rw-r--  1 bitnami daemon  3937 Aug 28  2024 wp-load.php

\-rw-rw-r--  1 bitnami daemon 51414 Jun  4 00:37 wp-login.php

\-rw-rw-r--  1 bitnami daemon  8727 Jun  4 00:37 wp-mail.php

\-rw-rw-r--  1 bitnami daemon 30081 Jun  4 00:37 wp-settings.php

\-rw-rw-r--  1 bitnami daemon 34516 Jun  4 00:37 wp-signup.php

\-rw-rw-r--  1 bitnami daemon  5102 Nov 15  2024 wp-trackback.php

\-rw-rw-r--  1 bitnami daemon  3205 Jun  4 00:37 xmlrpc.php

bitnami@ip-172-26-1-92:/opt/bitnami/wordpress$

\`\`\` |

Thank you. The output confirms that the staging site's folder is **not** in the WordPress root directory.

This is a good thing. It means the staging site was likely partially deleted or its record is corrupted within the plugin's settings, which is why the UI is hanging. We just need to find the database tables and force-delete them.

Let's check the database directly. This will be the definitive answer.

-----

### Step 1: Find the Staging Database Tables

Connect to your **old server (`18.221.253.166`)** via PuTTY and navigate to the WordPress root.

1.  **Go to the WordPress Directory:**

    ```bash
    cd /opt/bitnami/wordpress
    ```

2.  **List Staging Tables:** Run the following command to find any database tables created by WP Staging.

    ```bash
    sudo wp db tables --all-tables-with-prefix | grep 'wpstg'
    ```

-----

### Please provide the output of that command. There are two possible outcomes:

  * **Scenario A: The command lists tables** (e.g., `wpstg0_options`, `wpstg0_posts`, etc.).

      * This means the staging site's database still exists. If you provide the list, I will give you the precise command to drop all of them at once.

  * **Scenario B: The command returns nothing (no output).**

      * This is the best-case scenario. It means there are **no staging tables to delete**. The plugin is simply stuck on a "ghost" entry. The files are gone, the tables are gone, and you can proceed directly to deactivating the plugin.

Once we have this final piece of information, we can resolve the issue.


| User Prompt: |
|-------------|
| No tables returned. |

Excellent. That's the best possible outcome.

This confirms that the staging site is **completely gone**. The files have been deleted, and the database tables have been deleted.

The reason the plugin is "spinning" is because it's stuck on a "ghost" entry—a record of a staging site that no longer exists. The deletion script can't find the files or database tables it's looking for, so it hangs forever.

Since the manual cleanup is already done, we can simply force the deactivation of the plugin from the command line and then delete it.

-----

### The Final Step: Force Deactivation and Delete

1.  **Connect to Your OLD Server:** Make sure you are in your PuTTY session for `lexshift-old` (`18.221.253.166`).

2.  **Navigate to the WordPress Directory:**

    ```bash
    cd /opt/bitnami/wordpress
    ```

3.  **Force Deactivate the Plugins:** Run the following command to deactivate both the free and pro versions of the plugin at once.

    ```bash
    sudo wp plugin deactivate wp-staging wp-staging-pro
    ```

    You should see a success message for both.

4.  **Delete from the Dashboard:**

      * Now, log in to your old site's WordPress admin dashboard (`http://18.221.253.166/admin`).
      * Go to **Plugins -\> Installed Plugins**.
      * You will see that "WP Staging" and "WP Staging Pro" are now inactive.
      * You can now safely click the **Delete** link for both of them.

This will completely remove the plugins and fully resolve the issue.


| User Prompt: |
|-------------|
| Done. Now, I need to install the WP Staging plugin in the new site. I have the downloaded zip file from my account with WP Staging Pro. Please advise. |

Excellent. Installing WP Staging Pro is a great step for creating a safe development environment on your new site.

The installation process is straightforward, but it's important to know that WP Staging Pro works as an **add-on** to the free base plugin. You need to install both for it to function correctly.

Here is the step-by-step process.

***

### 🛡️ 1. Safety First: Backup Your New Site

Before installing any new plugin, especially one as powerful as a staging plugin, it's critical to have a fresh backup.

* **Action:** Go to your Amazon Lightsail dashboard and take a **manual snapshot** of your new `lexshift-new` instance (`3.150.250.227`). This ensures you have a perfect restore point.

***

### 🛠️ 2. The Two-Part Installation Process

You will perform these steps on your **new site's** dashboard (`https://lexshift.com/admin`).

#### Part A: Install the Free Base Plugin from the WordPress Repository

1.  In your WordPress dashboard, navigate to **Plugins -> Add New**.
2.  In the search bar, type **"WP Staging"**.
3.  Look for the plugin named "WP Staging – Backup Duplicator & Migration" by WP-STAGING.
4.  Click **Install Now**, and once it's finished, click **Activate**.

You now have the core plugin installed. The next step is to add your Pro features.

#### Part B: Upload and Install the WP Staging Pro Plugin

1.  From the same **Plugins -> Add New** screen, click the **"Upload Plugin"** button at the top of the page.
2.  Click the **"Choose File"** button and select the `.zip` file for WP Staging Pro that you downloaded from your account.
3.  Click **"Install Now"**.
4.  After the installation is complete, click **"Activate Plugin"**.

***

### ✅ 3. Activate Your License Key

The final step is to enter your license key to unlock all the professional features.

1.  In the left-hand menu of your WordPress dashboard, find the new **WP Staging** menu item.
2.  Click on it, then navigate to **Settings -> License**.
3.  Enter the license key you received from your WP Staging Pro account.
4.  Click **"Save Settings"** or **"Activate License"**.

Your new site is now equipped with a fully functional staging environment. You can proceed to create your first staging site whenever you're ready to test changes safely.


| User Prompt: |
|-------------|
| I have a large number (approx 80-100) font files that appear in my media library. Is this normal? |

No, that isn't typical for a standard WordPress site, but it is the **expected and normal behavior** for a specific feature you are likely using: **Elementor's Custom Fonts**.

This happens when you or a designer uses Elementor to upload custom font families (like `.woff`, `.ttf`, `.svg`, etc.) to ensure your site's typography matches your brand guidelines perfectly. Elementor stores these uploaded files directly in the Media Library.

* * *

### \## Where to Manage Your Fonts

The most important thing to know is that you should **never delete these fonts directly from the Media Library**. Doing so can break your site's design without removing the font from Elementor's settings, leading to errors.

Instead, you should always manage them from Elementor's dedicated font management screen.

1.  In your WordPress dashboard, navigate to **Elementor -> Custom Fonts**.
    
2.  You will see a list of all the custom font families that have been uploaded. Each family (e.g., "Helvetica Neue") will contain multiple font files for different weights and styles (bold, italic, regular, etc.).
    

* * *

### \## Action Plan: Review and Clean Up

Given the large number of files, it's a good idea to audit them and remove any that aren't part of your official brand identity.

#### 1\. **Identify Brand Fonts vs. Unused Fonts**

Your official brand guidelines specify **Helvetica Neue** for headlines and **Arial** for body text. On the **Elementor -> Custom Fonts** screen, review the list. Any font family that is *not* Helvetica Neue is likely a remnant from a theme demo, a design experiment, or an old brand guide and can probably be removed.

#### 2\. **Safely Delete Unused Font Families**

For any font family you wish to remove:

*   Hover over its name in the **Elementor -> Custom Fonts** list.
    
*   Click the red **"Delete"** link that appears.
    
*   Elementor will ask for confirmation. Proceed with the deletion.
    

✅ This is the correct method. Elementor will automatically remove all the associated font files (e.g., the `.woff`, `.ttf` files) from your Media Library for you.

#### 3\. **What NOT to Do**

Do not go into the **Media -> Library** and start deleting the font files one by one. This will break the connection in Elementor and can cause your site's text to render incorrectly. 🗑️

By managing your fonts through Elementor's interface, you ensure your site remains stable while keeping your Media Library clean and relevant.


| User Prompt: |
|-------------|
| Can I run a list of everything in my media library from the command prompt, then have you review for anything out of ordinary? |

Yes, you can. The best way to get a clean, comprehensive list of your media library is by using a WP-CLI command.

* * *

### \## Command to List Media Library Items

Connect to your **new server (`3.150.250.227`)** via PuTTY or your preferred SSH client and run the following commands.

1.  **Navigate to the WordPress Directory:**
    
    Bash
    
    ```
    cd /opt/bitnami/wordpress
    ```
    
2.  **Run the List Command:** This command will generate a table of all attachments, showing their ID, title, file type, and direct URL.
    
    Bash
    
    ```
    wp post list --post_type=attachment --fields=ID,post_title,post_mime_type,guid --posts_per_page=-1
    ```
    *   `--post_type=attachment`: This is the key part that tells WordPress to only list items from the Media Library.
        
    *   `--fields=...`: This specifies which columns to show in the output.
        
    *   `--posts_per_page=-1`: This ensures the command lists *all* items without cutting the list short.
        

* * *

### \## How to Proceed

Simply copy the entire table that the command outputs and paste it back here. I will review the list for anything unusual, such as unexpected file types (`.php`, `.js`), strange filenames, or incorrect URL paths.


| User Prompt: |
|-------------|
| Please advise:

  

\`\`\`bash

  

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp post list --post\_type=attachment --fields=ID,post\_title,post\_mime\_type,guid --posts\_per\_page=-1

+------+---------------------------------------------+-------------------------------+----------------------------------------------+

| ID   | post\_title                                  | post\_mime\_type                | guid                                         |

+------+---------------------------------------------+-------------------------------+----------------------------------------------+

| 2229 | orchestrate-light                           | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/orchestrate-light.svg                    |

| 2230 | sharer\_dark\_logo                            | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/sharer\_dark\_logo.svg                     |

| 2231 | sharer\_icon\_dark                            | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/sharer\_icon\_dark.svg                     |

| 2232 | sharer\_icon\_light                           | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/sharer\_icon\_light.svg                    |

| 2233 | sharer\_light\_logo                           | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/sharer\_light\_logo.svg                    |

| 2234 | sharer\_vertical\_dark                        | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/sharer\_vertical\_dark.svg                 |

| 2235 | sharer\_vertical\_light                       | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/sharer\_vertical\_light.svg                |

| 2218 | iQ-dark                                     | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/iQ-dark.svg                              |

| 2219 | iQ-light                                    | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/iQ-light.svg                             |

| 2220 | illuminate-dark                             | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/illuminate-dark.svg                      |

| 2221 | illuminate-light                            | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/illuminate-light.svg                     |

| 2222 | lexshift-graphic-dark                       | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/lexshift-graphic-dark.svg                |

| 2223 | lexshift-graphic-light                      | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/lexshift-graphic-light.svg               |

| 2224 | lexshift-horiz-dark                         | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/lexshift-horiz-dark.svg                  |

| 2225 | lexshift-horiz-light                        | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/lexshift-horiz-light.svg                 |

| 2226 | mosaIQ-dark                                 | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/mosaIQ-dark.svg                          |

| 2227 | mosaIQ-light                                | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/mosaIQ-light.svg                         |

| 2228 | orchestrate-dark                            | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/orchestrate-dark.svg                     |

| 2217 | booth-rear-counter\_rgb\_4096w                | image/png                     | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/booth-rear-counter\_rgb\_4096w.png         |

| 2215 | booth-header\_rgb\_4096w                      | image/png                     | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/booth-header\_rgb\_4096w.png               |

| 2216 | booth-rear-counter                          | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/booth-rear-counter.svg                   |

| 2210 | Bkgrnd-paralax2                             | image/webp                    | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/Bkgrnd-paralax2.webp                     |

| 2211 | booth-backwall                              | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/booth-backwall.svg                       |

| 2212 | booth-backwall\_rgb\_4096w                    | image/png                     | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/booth-backwall\_rgb\_4096w.png             |

| 2213 | booth-front-counter                         | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/booth-front-counter.svg                  |

| 2214 | booth-header                                | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /10/booth-header.svg                         |

| 2328 | docs                                        | application/json              | https://lexshift.com/wp-content/uploads/wpfo |

|      |                                             |                               | rms/cache/docs.json                          |

| 2333 | templates                                   | text/plain                    | https://lexshift.com/wp-content/uploads/wpfo |

|      |                                             |                               | rms/cache/templates.json                     |

| 2327 | addons                                      | application/json              | https://lexshift.com/wp-content/uploads/wpfo |

|      |                                             |                               | rms/cache/addons.json                        |

| 2332 | splash                                      | application/json              | https://lexshift.com/wp-content/uploads/wpfo |

|      |                                             |                               | rms/cache/splash.json                        |

| 2334 | themes-custom                               | application/json              | https://lexshift.com/wp-content/uploads/wpfo |

|      |                                             |                               | rms/themes/themes-custom.json                |

| 2115 | phone-icon                                  | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/phone-icon.png                    |

| 2113 | location-icon                               | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/location-icon.png                 |

| 2114 | email-icon                                  | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/email-icon.png                    |

| 2066 | john-announcement                           | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/john-announcement.png             |

| 2055 | hex-O-illuminate-350w                       | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/hex-O-illuminate-350w.png         |

| 2051 | hex-O-orchestrate-350w                      | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/hex-O-orchestrate-350w.png        |

| 2018 | Bkgrnd-paralax3                             | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/Bkgrnd-paralax3.png               |

| 2000 | inline\_orchestrate                          | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/inline\_orchestrate.png            |

| 2001 | inline\_illuminate                           | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/inline\_illuminate.png             |

| 1993 | just\_illuminate2                            | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/just\_illuminate2.png              |

| 1994 | just\_orchestrate2                           | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/just\_orchestrate2.png             |

| 1987 | just\_orchestrate                            | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/just\_orchestrate.png              |

| 1986 | just\_illuminate                             | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/just\_illuminate.png               |

| 1965 | orchestrate-dark\_rgb\_128w                   | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/orchestrate-dark\_rgb\_128w.png     |

| 1964 | illuminate-dark\_rgb\_128w                    | image/png                     | http://www.new.lexshift.com/wp-content/uploa |

|      |                                             |                               | ds/2025/10/illuminate-dark\_rgb\_128w.png      |

| 2325 | onepress-d1260f352d3fc21905bbf32d360c22bd   | text/plain                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/onepress-d1260f352d3fc2190 |

|      |                                             |                               | 5bbf32d360c22bd.css                          |

| 2329 | email-summaries                             | application/json              | https://lexshift.com/wp-content/uploads/wpfo |

|      |                                             |                               | rms/cache/email-summaries.json               |

| 1906 | sharer\_dark\_logo\_rgb\_512w.webp              | image/webp                    | https://18.222.78.21/sharer\_dark\_logo\_rgb\_51 |

|      |                                             |                               | 2w-webp/                                     |

| 1907 | sharer\_dark\_logo\_rgb\_512w                   | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/sharer\_dark\_logo\_rgb\_512w.png             |

| 1892 | WesternIM.Logo\_.Master.webp                 | image/webp                    | https://18.222.78.21/westernim-logo\_-master- |

|      |                                             |                               | webp/                                        |

| 1893 | WesternIM.Logo\_.Master                      | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/WesternIM.Logo\_.Master.png                |

| 1888 | Logo-with-Side-name1.webp                   | image/webp                    | https://18.222.78.21/logo-with-side-name1-we |

|      |                                             |                               | bp/                                          |

| 1889 | Logo-with-Side-name1                        | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/Logo-with-Side-name1.png                  |

| 2330 | 4.18.0,277742924da1d9c699129743a1fff942661a | application/octet-stream      | https://lexshift.com/wp-content/uploads/wpfo |

|      | 388d,1                                      |                               | rms/cache/htmlpurifier/HTML/4.18.0,277742924 |

|      |                                             |                               | da1d9c699129743a1fff942661a388d,1.ser        |

| 1876 | john\_aanouncement.webp                      | image/webp                    | https://18.222.78.21/john\_aanouncement-webp/ |

| 1877 | john\_aanouncement                           | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/john\_aanouncement.png                     |

| 1860 | MBPFP.webp                                  | image/webp                    | https://18.222.78.21/mbpfp-webp/             |

| 1861 | MBPFP                                       | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/MBPFP.png                                 |

| 1850 | vision.webp                                 | image/webp                    | https://18.222.78.21/vision-webp/            |

| 1851 | vision                                      | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/vision.png                                |

| 1849 | mission                                     | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/mission.png                               |

| 1848 | mission.webp                                | image/webp                    | https://18.222.78.21/mission-webp/           |

| 1813 | Andrew Borodenko                            | image/webp                    | https://18.222.78.21/hex-andyv2-webp/        |

| 1814 | Andrew Borodenko                            | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-andyV2.png                            |

| 1805 | Jeffrey C. Sharer                           | image/webp                    | https://18.222.78.21/hex-jeff-webp/          |

| 1806 | Jeffrey C. Sharer                           | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-jeff.png                              |

| 1803 | Jason C. Stearns                            | image/webp                    | https://18.222.78.21/hex-jason-webp/         |

| 1804 | Jason C. Stearns                            | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-jason.png                             |

| 1801 | Terry Wong                                  | image/webp                    | https://18.222.78.21/hex-terry-webp/         |

| 1802 | Terry Wong                                  | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-terry.png                             |

| 1799 | Shirley Qin                                 | image/webp                    | https://18.222.78.21/hex-shirley-webp/       |

| 1800 | Shirley Qin                                 | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-shirley.png                           |

| 1795 | John J. Isaza                               | image/webp                    | https://18.222.78.21/hex-john-webp/          |

| 1796 | John J Isaza                                | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-john.png                              |

| 1654 | lexshift-vert-dark\_rgb\_256w-AB              | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/lexshift-vert-dark\_rgb\_256w-AB.png        |

| 1653 | lexshift-vert-dark\_rgb\_256w-AB.webp         | image/webp                    | https://18.222.78.21/lexshift-vert-dark\_rgb\_ |

|      |                                             |                               | 256w-ab-webp/                                |

| 1648 | IsazaLaw-308h                               | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/IsazaLaw-308h.png                         |

| 1647 | IsazaLaw-308h.webp                          | image/webp                    | https://18.222.78.21/isazalaw-308h-webp/     |

| 1646 | lexshift-vert-dark\_rgb\_256w                 | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/lexshift-vert-dark\_rgb\_256w.png           |

| 1645 | lexshift-vert-dark\_rgb\_256w.webp            | image/webp                    | https://18.222.78.21/lexshift-vert-dark\_rgb\_ |

|      |                                             |                               | 256w-webp/                                   |

| 1643 | lexshift-horiz-dark\_rgb\_256w.webp           | image/webp                    | https://18.222.78.21/lexshift-horiz-dark\_rgb |

|      |                                             |                               | \_256w-webp-2/                                |

| 1641 | IsazaLaw-256w.webp                          | image/webp                    | https://18.222.78.21/isazalaw-256w-webp/     |

| 1635 | hex-illuminate-350w                         | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-illuminate-350w.png                   |

| 1634 | hex-illuminate-350w.webp                    | image/webp                    | https://18.222.78.21/hex-illuminate-350w-web |

|      |                                             |                               | p/                                           |

| 1633 | hex-orchestrate-350w                        | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-orchestrate-350w.png                  |

| 1632 | hex-orchestrate-350w.webp                   | image/webp                    | https://18.222.78.21/hex-orchestrate-350w-we |

|      |                                             |                               | bp/                                          |

| 1630 | hex-orchestrate-256w                        | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-orchestrate-256w.png                  |

| 1629 | hex-orchestrate-256w.webp                   | image/webp                    | https://18.222.78.21/hex-orchestrate-256w-we |

|      |                                             |                               | bp/                                          |

| 1622 | hex-illuminate-512w                         | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-illuminate-512w.png                   |

| 1621 | hex-illuminate-512w.webp                    | image/webp                    | https://18.222.78.21/hex-illuminate-512w-web |

|      |                                             |                               | p/                                           |

| 1620 | hex-orchestrate-512w                        | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/hex-orchestrate-512w.png                  |

| 1619 | hex-orchestrate-512w.webp                   | image/webp                    | https://18.222.78.21/hex-orchestrate-512w-we |

|      |                                             |                               | bp/                                          |

| 1594 | orchestrate-dark\_rgb\_512w-AB                | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 10/orchestrate-dark\_rgb\_512w-AB.png          |

| 1593 | orchestrate-dark\_rgb\_512w-AB.webp           | image/webp                    | https://18.222.78.21/orchestrate-dark\_rgb\_51 |

|      |                                             |                               | 2w-ab-webp/                                  |

| 2321 | onepress-3b44af6a3d31d481ea45d1476c40cf80   | text/plain                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/onepress-3b44af6a3d31d481e |

|      |                                             |                               | a45d1476c40cf80.css                          |

| 2322 | onepress-4ae0c19ad0b33afa239789354de2f837   | text/plain                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/onepress-4ae0c19ad0b33afa2 |

|      |                                             |                               | 39789354de2f837.css                          |

| 1556 | Bkgrnd-paralax2                             | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/Bkgrnd-paralax2.png                       |

| 1555 | Bkgrnd-paralax2.webp                        | image/webp                    | https://18.222.78.21/bkgrnd-paralax2-webp/   |

| 1552 | orchestrate-dark\_rgb\_256w-AB.webp           | image/webp                    | https://18.222.78.21/orchestrate-dark\_rgb\_25 |

|      |                                             |                               | 6w-ab-webp/                                  |

| 1553 | orchestrate-dark\_rgb\_256w-AB                | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/orchestrate-dark\_rgb\_256w-AB.png          |

| 1543 | quickbase-300x75-1.webp                     | image/webp                    | https://18.222.78.21/quickbase-300x75-1-webp |

|      |                                             |                               | /                                            |

| 1544 | quickbase-300x75                            | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/quickbase-300x75-1.png                    |

| 1541 | cydeas-300x79-1.webp                        | image/webp                    | https://18.222.78.21/cydeas-300x79-1-webp/   |

| 1542 | cydeas-300x79                               | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/cydeas-300x79-1.png                       |

| 1539 | insight-optix-300x79-1.webp                 | image/webp                    | https://18.222.78.21/insight-optix-300x79-1- |

|      |                                             |                               | webp/                                        |

| 1540 | insight-optix-300x79                        | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/insight-optix-300x79-1.png                |

| 1537 | neota-300x80-1.webp                         | image/webp                    | https://18.222.78.21/neota-300x80-1-webp/    |

| 1538 | neota-300x80                                | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/neota-300x80-1.png                        |

| 1533 | microsoft-300x75-1.webp                     | image/webp                    | https://18.222.78.21/microsoft-300x75-1-webp |

|      |                                             |                               | /                                            |

| 1534 | microsoft-300x75                            | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/microsoft-300x75-1.png                    |

| 1531 | Filerskeepers-300x75-1.webp                 | image/webp                    | https://18.222.78.21/filerskeepers-300x75-1- |

|      |                                             |                               | webp/                                        |

| 1532 | Filerskeepers-300x75                        | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/Filerskeepers-300x75-1.png                |

| 1529 | lexfusion-300x80-1.webp                     | image/webp                    | https://18.222.78.21/lexfusion-300x80-1-webp |

|      |                                             |                               | /                                            |

| 1530 | lexfusion-300x80                            | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/lexfusion-300x80-1.png                    |

| 1502 | illuminate-dark\_rgb\_256w                    | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/illuminate-dark\_rgb\_256w.png              |

| 1501 | illuminate-dark\_rgb\_256w.webp               | image/webp                    | https://18.222.78.21/illuminate-dark\_rgb\_256 |

|      |                                             |                               | w-webp/                                      |

| 2323 | onepress-78de2ac528e30804442626c2d81e6701   | text/plain                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/onepress-78de2ac528e308044 |

|      |                                             |                               | 42626c2d81e6701.css                          |

| 2238 | Poppins-v24-pxiEyp8kv8JHgFVrJJfedQ          | image/svg+xml                 | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/Poppins-v24-pxiEyp8k |

|      |                                             |                               | v8JHgFVrJJfedQ.svg                           |

| 2254 | https:--fonts.gstatic                       | application/vnd.ms-fontobject | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/https:--fonts.gstati |

|      |                                             |                               | c.com-l-font?kit=pxiEyp8kv8JHgFVrJJfedg&amp; |

|      |                                             |                               | skey=87759fb096548f6d&amp;v=v24              |

| 2289 | poppins-v24-pxiByp8kv8JHgFVrLGT9Z11lFd2JQEl | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 8qw                                         |                               | le-fonts/onepress/fonts/poppins-v24-pxiByp8k |

|      |                                             |                               | v8JHgFVrLGT9Z11lFd2JQEl8qw.woff2             |

| 2290 | poppins-v24-pxiByp8kv8JHgFVrLGT9Z1JlFd2JQEl | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 8qw                                         |                               | le-fonts/onepress/fonts/poppins-v24-pxiByp8k |

|      |                                             |                               | v8JHgFVrLGT9Z1JlFd2JQEl8qw.woff2             |

| 2291 | poppins-v24-pxiByp8kv8JHgFVrLGT9Z1xlE92JQEk | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiByp8k |

|      |                                             |                               | v8JHgFVrLGT9Z1xlE92JQEk.woff                 |

| 2292 | poppins-v24-pxiByp8kv8JHgFVrLGT9Z1xlEN2JQEk | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiByp8k |

|      |                                             |                               | v8JHgFVrLGT9Z1xlEN2JQEk.ttf                  |

| 2293 | poppins-v24-pxiByp8kv8JHgFVrLGT9Z1xlFd2JQEk | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiByp8k |

|      |                                             |                               | v8JHgFVrLGT9Z1xlFd2JQEk.woff2                |

| 2294 | poppins-v24-pxiEyp8kv8JHgFVrJJbecnFHGPezSQ  | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiEyp8k |

|      |                                             |                               | v8JHgFVrJJbecnFHGPezSQ.woff2                 |

| 2295 | poppins-v24-pxiEyp8kv8JHgFVrJJfecnFHGPc     | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiEyp8k |

|      |                                             |                               | v8JHgFVrJJfecnFHGPc.woff2                    |

| 2296 | poppins-v24-pxiEyp8kv8JHgFVrJJfed3FHGPc     | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiEyp8k |

|      |                                             |                               | v8JHgFVrJJfed3FHGPc.ttf                      |

| 2297 | poppins-v24-pxiEyp8kv8JHgFVrJJfedHFHGPc     | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiEyp8k |

|      |                                             |                               | v8JHgFVrJJfedHFHGPc.woff                     |

| 2298 | poppins-v24-pxiEyp8kv8JHgFVrJJnecnFHGPezSQ  | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/poppins-v24-pxiEyp8k |

|      |                                             |                               | v8JHgFVrJJnecnFHGPezSQ.woff2                 |

| 1495 | illuminate-dark\_rgb\_512w                    | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/illuminate-dark\_rgb\_512w.png              |

| 1494 | illuminate-dark\_rgb\_512w.webp               | image/webp                    | https://18.222.78.21/illuminate-dark\_rgb\_512 |

|      |                                             |                               | w-webp/                                      |

| 1492 | orchestrate-dark\_rgb\_512w                   | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/orchestrate-dark\_rgb\_512w.png             |

| 1491 | orchestrate-dark\_rgb\_512w.webp              | image/webp                    | https://18.222.78.21/orchestrate-dark\_rgb\_51 |

|      |                                             |                               | 2w-webp/                                     |

| 1489 | orchestrate-dark\_rgb\_256w.webp              | image/webp                    | https://18.222.78.21/orchestrate-dark\_rgb\_25 |

|      |                                             |                               | 6w-webp/                                     |

| 1473 | mosaIQ-dark\_rgb\_512w-A                      | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/mosaIQ-dark\_rgb\_512w-A.png                |

| 1472 | mosaIQ-dark\_rgb\_512w-A.webp                 | image/webp                    | https://18.222.78.21/mosaiq-dark\_rgb\_512w-a- |

|      |                                             |                               | webp/                                        |

| 2320 | onepress-2a084129029d55f2db83f4a3803de22b   | text/plain                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/onepress-2a084129029d55f2d |

|      |                                             |                               | b83f4a3803de22b.css                          |

| 2250 | http:--fonts.gstatic                        | application/vnd.ms-fontobject | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/http:--fonts.gstatic |

|      |                                             |                               | .com-l-font?kit=1Pt\_g8LJRfWJmhDAuUsSQamb1W0l |

|      |                                             |                               | wk4S4WjMPrU&amp;skey=45c59603efbe20e8&amp;v= |

|      |                                             |                               | v47                                          |

| 2324 | onepress-cbaee44331079932e6badb6b2a13a270   | text/plain                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/onepress-cbaee44331079932e |

|      |                                             |                               | 6badb6b2a13a270.css                          |

| 2236 | Comfortaa-v47-1Pt\_g8LJRfWJmhDAuUsSQamb1W0lw | image/svg+xml                 | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMPrY                                  |                               | le-fonts/onepress/fonts/Comfortaa-v47-1Pt\_g8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMPrY.svg        |

| 2240 | comfortaa-v47-1PtCg8LJRfWJmhDAuUsSQamb1W0lw | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMXL830efAesmwYSFoxBEA\_I28dnyMpxo      |                               | le-fonts/onepress/fonts/comfortaa-v47-1PtCg8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMXL830efAesmwYS |

|      |                                             |                               | FoxBEA\_I28dnyMpxo.woff2                      |

| 2241 | comfortaa-v47-1PtCg8LJRfWJmhDAuUsSQamb1W0lw | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMXL830efAesmwYSFoxBEB\_I28dnyMpxo      |                               | le-fonts/onepress/fonts/comfortaa-v47-1PtCg8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMXL830efAesmwYS |

|      |                                             |                               | FoxBEB\_I28dnyMpxo.woff2                      |

| 2244 | comfortaa-v47-1PtCg8LJRfWJmhDAuUsSQamb1W0lw | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMXL830efAesmwYSFoxBEM\_I28dnyMpxo      |                               | le-fonts/onepress/fonts/comfortaa-v47-1PtCg8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMXL830efAesmwYS |

|      |                                             |                               | FoxBEM\_I28dnyMpxo.woff2                      |

| 2245 | comfortaa-v47-1PtCg8LJRfWJmhDAuUsSQamb1W0lw | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMXL830efAesmwYSFoxBEP\_I28dnyM         |                               | le-fonts/onepress/fonts/comfortaa-v47-1PtCg8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMXL830efAesmwYS |

|      |                                             |                               | FoxBEP\_I28dnyM.woff2                         |

| 2246 | comfortaa-v47-1Pt\_g8LJRfWJmhDAuUsSQamb1W0lw | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMPrQVIT9d4cw                          |                               | le-fonts/onepress/fonts/comfortaa-v47-1Pt\_g8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMPrQVIT9d4cw.tt |

|      |                                             |                               | f                                            |

| 2247 | comfortaa-v47-1Pt\_g8LJRfWJmhDAuUsSQamb1W0lw | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMPrcVJz9d                             |                               | le-fonts/onepress/fonts/comfortaa-v47-1Pt\_g8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMPrcVJz9d.woff  |

| 2248 | comfortaa-v47-1Pt\_g8LJRfWJmhDAuUsSQamb1W0lw | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4Y\_LPrQVIT9d4cw                          |                               | le-fonts/onepress/fonts/comfortaa-v47-1Pt\_g8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4Y\_LPrQVIT9d4cw.tt |

|      |                                             |                               | f                                            |

| 2249 | comfortaa-v47-1Pt\_g8LJRfWJmhDAuUsSQamb1W0lw | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4Y\_LPrcVJz9d                             |                               | le-fonts/onepress/fonts/comfortaa-v47-1Pt\_g8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4Y\_LPrcVJz9d.woff  |

| 2251 | https:--fonts.gstatic                       | application/vnd.ms-fontobject | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/https:--fonts.gstati |

|      |                                             |                               | c.com-l-font?kit=1Pt\_g8LJRfWJmhDAuUsSQamb1W0 |

|      |                                             |                               | lwk4S4WjMPrU&amp;skey=45c59603efbe20e8&amp;v |

|      |                                             |                               | =v47                                         |

| 2304 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVs | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | 9pbCGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVs9pbCGPrQVIT9d4cw.ttf  |

| 2306 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVs | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | EpbCGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVsEpbCGPrQVIT9d4cw.ttf  |

| 2308 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVt | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | apbCGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVtapbCGPrQVIT9d4cw.ttf  |

| 2310 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVt | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | zpbCGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVtzpbCGPrQVIT9d4cw.ttf  |

| 2326 | onepress-f48c9a05f6d2a0e715f599dc048a4391   | text/plain                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/onepress-f48c9a05f6d2a0e71 |

|      |                                             |                               | 5f599dc048a4391.css                          |

| 2237 | OpenSans-v44-memSYaGs126MiZpBA-UvWbX2vVnXBb | image/svg+xml                 | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVZyOOSr4dVJWUgsjZ0B4uaVU               |                               | le-fonts/onepress/fonts/OpenSans-v44-memSYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVZyOOSr4dVJWUgsj |

|      |                                             |                               | Z0B4uaVU.svg                                 |

| 2239 | Raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVv | image/svg+xml                 | https://lexshift.com/wp-content/uploads/goog |

|      | aorCGPrY                                    |                               | le-fonts/onepress/fonts/Raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVvaorCGPrY.svg          |

| 2242 | comfortaa-v47-1PtCg8LJRfWJmhDAuUsSQamb1W0lw | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMXL830efAesmwYSFoxBEC\_I28dnyMpxo      |                               | le-fonts/onepress/fonts/comfortaa-v47-1PtCg8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMXL830efAesmwYS |

|      |                                             |                               | FoxBEC\_I28dnyMpxo.woff2                      |

| 2243 | comfortaa-v47-1PtCg8LJRfWJmhDAuUsSQamb1W0lw | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | k4S4WjMXL830efAesmwYSFoxBEL\_I28dnyMpxo      |                               | le-fonts/onepress/fonts/comfortaa-v47-1PtCg8 |

|      |                                             |                               | LJRfWJmhDAuUsSQamb1W0lwk4S4WjMXL830efAesmwYS |

|      |                                             |                               | FoxBEL\_I28dnyMpxo.woff2                      |

| 2252 | https:--fonts.gstatic                       | application/vnd.ms-fontobject | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/https:--fonts.gstati |

|      |                                             |                               | c.com-l-font?kit=1Ptxg8zYS\_SKggPN4iEgvnHyvve |

|      |                                             |                               | LxVvaorCGPrU&amp;skey=30a27f2564731c64&amp;v |

|      |                                             |                               | =v37                                         |

| 2253 | https:--fonts.gstatic                       | application/vnd.ms-fontobject | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/https:--fonts.gstati |

|      |                                             |                               | c.com-l-font?kit=memSYaGs126MiZpBA-UvWbX2vVn |

|      |                                             |                               | XBbObj2OVZyOOSr4dVJWUgsjZ0B4uaVY&amp;skey=62 |

|      |                                             |                               | c1cbfccc78b4b2&amp;v=v44                     |

| 2312 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVu | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | EorCGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVuEorCGPrQVIT9d4cw.ttf  |

| 2314 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVv | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | ao7CGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVvao7CGPrQVIT9d4cw.ttf  |

| 2316 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVv | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | aorCGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVvaorCGPrQVIT9d4cw.ttf  |

| 2318 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVv | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | oorCGPrQVIT9d4cw                            |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVvoorCGPrQVIT9d4cw.ttf  |

| 2264 | opensans-v44-memSYaGs126MiZpBA-UvWbX2vVnXBb | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVZyOOSr4dVJWUgsgH1x4uaVcUwaEQXjM       |                               | le-fonts/onepress/fonts/opensans-v44-memSYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVZyOOSr4dVJWUgsg |

|      |                                             |                               | H1x4uaVcUwaEQXjM.ttf                         |

| 2266 | opensans-v44-memSYaGs126MiZpBA-UvWbX2vVnXBb | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVZyOOSr4dVJWUgsiH0B4uaVcUwaEQXjM       |                               | le-fonts/onepress/fonts/opensans-v44-memSYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVZyOOSr4dVJWUgsi |

|      |                                             |                               | H0B4uaVcUwaEQXjM.ttf                         |

| 2268 | opensans-v44-memSYaGs126MiZpBA-UvWbX2vVnXBb | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVZyOOSr4dVJWUgsjZ0B4uaVcUwaEQXjM       |                               | le-fonts/onepress/fonts/opensans-v44-memSYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVZyOOSr4dVJWUgsj |

|      |                                             |                               | Z0B4uaVcUwaEQXjM.ttf                         |

| 2305 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVs | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | 9pbCGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVs9pbCGPrcVJz9d.woff    |

| 2307 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVs | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | EpbCGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVsEpbCGPrcVJz9d.woff    |

| 2309 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVt | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | apbCGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVtapbCGPrcVJz9d.woff    |

| 2311 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVt | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | zpbCGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVtzpbCGPrcVJz9d.woff    |

| 2313 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVu | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | EorCGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVuEorCGPrcVJz9d.woff    |

| 2315 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVv | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | ao7CGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVvao7CGPrcVJz9d.woff    |

| 2317 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVv | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | aorCGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVvaorCGPrcVJz9d.woff    |

| 2319 | raleway-v37-1Ptxg8zYS\_SKggPN4iEgvnHyvveLxVv | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | oorCGPrcVJz9d                               |                               | le-fonts/onepress/fonts/raleway-v37-1Ptxg8zY |

|      |                                             |                               | S\_SKggPN4iEgvnHyvveLxVvoorCGPrcVJz9d.woff    |

| 2255 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0Rk5hkWV4exYMUdjFnmg    |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | Rk5hkWV4exYMUdjFnmg.ttf                      |

| 2257 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0Rk8ZkWV4exYMUdjFnmg    |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | Rk8ZkWV4exYMUdjFnmg.ttf                      |

| 2259 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0RkxhjWV4exYMUdjFnmg    |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | RkxhjWV4exYMUdjFnmg.ttf                      |

| 2260 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0RkxhjWV4exoMSdjE       |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | RkxhjWV4exoMSdjE.woff                        |

| 2261 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/sfnt                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0RkyFjWV4exYMUdjFnmg    |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | RkyFjWV4exYMUdjFnmg.ttf                      |

| 2262 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0RkyFjWV4exoMSdjE       |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | RkyFjWV4exoMSdjE.woff                        |

| 2263 | opensans-v44-memSYaGs126MiZpBA-UvWbX2vVnXBb | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVZyOOSr4dVJWUgsgH1x4uaVQUx6EQ          |                               | le-fonts/onepress/fonts/opensans-v44-memSYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVZyOOSr4dVJWUgsg |

|      |                                             |                               | H1x4uaVQUx6EQ.woff                           |

| 2265 | opensans-v44-memSYaGs126MiZpBA-UvWbX2vVnXBb | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVZyOOSr4dVJWUgsiH0B4uaVQUx6EQ          |                               | le-fonts/onepress/fonts/opensans-v44-memSYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVZyOOSr4dVJWUgsi |

|      |                                             |                               | H0B4uaVQUx6EQ.woff                           |

| 2267 | opensans-v44-memSYaGs126MiZpBA-UvWbX2vVnXBb | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVZyOOSr4dVJWUgsjZ0B4uaVQUx6EQ          |                               | le-fonts/onepress/fonts/opensans-v44-memSYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVZyOOSr4dVJWUgsj |

|      |                                             |                               | Z0B4uaVQUx6EQ.woff                           |

| 2256 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0Rk5hkWV4exoMSdjE       |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | Rk5hkWV4exoMSdjE.woff                        |

| 2258 | opensans-v44-memQYaGs126MiZpBA-UFUIcVXSCEkx | font/woff                     | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWq8tWZ0Pw86hd0Rk8ZkWV4exoMSdjE       |                               | le-fonts/onepress/fonts/opensans-v44-memQYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWq8tWZ0Pw86hd0 |

|      |                                             |                               | Rk8ZkWV4exoMSdjE.woff                        |

| 2269 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqW106FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqW106FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2270 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWt06FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWt06FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2273 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWtk6FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWtk6FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2274 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWu06FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWu06FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2275 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWuU6FxZCJgg                        |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWuU6FxZCJgg. |

|      |                                             |                               | woff2                                        |

| 2276 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWuk6FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWuk6FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2278 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWxU6FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWxU6FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2279 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTS-mu0SC55I                           |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTS-mu0SC55I.wof |

|      |                                             |                               | f2                                           |

| 2280 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTS2mu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTS2mu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2281 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTSCmu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTSCmu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2282 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTSGmu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTSGmu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2283 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTSKmu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTSKmu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2284 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTSOmu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTSOmu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2285 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTSumu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTSumu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2286 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTSymu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTSymu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2287 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTUGmu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTUGmu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2288 | opensans-v44-memvYaGs126MiZpBA-UvWbX2vVnXBb | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | Obj2OVTVOmu0SC55K5gw                        |                               | le-fonts/onepress/fonts/opensans-v44-memvYaG |

|      |                                             |                               | s126MiZpBA-UvWbX2vVnXBbObj2OVTVOmu0SC55K5gw. |

|      |                                             |                               | woff2                                        |

| 2299 | raleway-v37-1Ptug8zYS\_SKggPNyC0IT4ttDfA     | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/raleway-v37-1Ptug8zY |

|      |                                             |                               | S\_SKggPNyC0IT4ttDfA.woff2                    |

| 2300 | raleway-v37-1Ptug8zYS\_SKggPNyCAIT4ttDfCmxA  | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/raleway-v37-1Ptug8zY |

|      |                                             |                               | S\_SKggPNyCAIT4ttDfCmxA.woff2                 |

| 2301 | raleway-v37-1Ptug8zYS\_SKggPNyCIIT4ttDfCmxA  | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/raleway-v37-1Ptug8zY |

|      |                                             |                               | S\_SKggPNyCIIT4ttDfCmxA.woff2                 |

| 2302 | raleway-v37-1Ptug8zYS\_SKggPNyCMIT4ttDfCmxA  | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/raleway-v37-1Ptug8zY |

|      |                                             |                               | S\_SKggPNyCMIT4ttDfCmxA.woff2                 |

| 2303 | raleway-v37-1Ptug8zYS\_SKggPNyCkIT4ttDfCmxA  | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      |                                             |                               | le-fonts/onepress/fonts/raleway-v37-1Ptug8zY |

|      |                                             |                               | S\_SKggPNyCkIT4ttDfCmxA.woff2                 |

| 2271 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWtE6FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWtE6FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2272 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWtU6FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWtU6FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 2277 | opensans-v44-memtYaGs126MiZpBA-UFUIcVXSCEkx | font/woff2                    | https://lexshift.com/wp-content/uploads/goog |

|      | 2cmqvXlWqWvU6FxZCJgvAQ                      |                               | le-fonts/onepress/fonts/opensans-v44-memtYaG |

|      |                                             |                               | s126MiZpBA-UFUIcVXSCEkx2cmqvXlWqWvU6FxZCJgvA |

|      |                                             |                               | Q.woff2                                      |

| 1434 | Bkgrnd-paralax                              | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/Bkgrnd-paralax-2.png                      |

| 1433 | Bkgrnd-paralax-2.webp                       | image/webp                    | https://18.222.78.21/bkgrnd-paralax-2-webp/  |

| 1416 | 1x1.webp                                    | image/webp                    | https://18.222.78.21/1x1-webp/               |

| 1417 | 1x1                                         | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/1x1.png                                   |

| 1413 | mosaIQ-dark\_rgb\_512w                        | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/mosaIQ-dark\_rgb\_512w.png                  |

| 1412 | mosaIQ-dark\_rgb\_512w.webp                   | image/webp                    | https://18.222.78.21/mosaiq-dark\_rgb\_512w-we |

|      |                                             |                               | bp/                                          |

| 1408 | Bkgrnd-paralax-1.webp                       | image/webp                    | https://18.222.78.21/bkgrnd-paralax-1-webp/  |

| 1406 | Bkgrnd-paralax                              | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/Bkgrnd-paralax.png                        |

| 1405 | Bkgrnd-paralax.webp                         | image/webp                    | https://18.222.78.21/bkgrnd-paralax-webp/    |

| 1399 | Bkgrnd.webp                                 | image/webp                    | https://18.222.78.21/bkgrnd-webp/            |

| 1384 | Background.webp                             | image/webp                    | https://18.222.78.21/background-webp/        |

| 1377 | Lexshift 512x512                            | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/cropped-Lexshift-512x512-1.png            |

| 1376 | Lexshift 512x512                            | image/png                     | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/Lexshift-512x512-1.png                    |

| 1375 | Lexshift-512x512-1.webp                     | image/webp                    | https://18.222.78.21/lexshift-512x512-1-webp |

|      |                                             |                               | /                                            |

| 1369 | lexshift-horiz-dark\_rgb\_128w.webp           | image/webp                    | https://18.222.78.21/lexshift-horiz-dark\_rgb |

|      |                                             |                               | \_128w-webp/                                  |

| 1367 | lexshift-horiz-dark\_rgb\_256w.webp           | image/webp                    | https://18.222.78.21/lexshift-horiz-dark\_rgb |

|      |                                             |                               | \_256w-webp/                                  |

| 1365 | lexshift-graphic-dark\_rgb\_256w.webp         | image/webp                    | https://18.222.78.21/lexshift-graphic-dark\_r |

|      |                                             |                               | gb\_256w-webp/                                |

| 1362 | onepress-demos-oneify-config                |                               | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/onepress-demos-oneify-config.json         |

| 1361 | onepress-demos-oneify-dummy-data            |                               | http://18.222.78.21/wp-content/uploads/2025/ |

|      |                                             |                               | 09/onepress-demos-oneify-dummy-data.xml      |

| 2331 | index                                       | application/x-empty           | https://lexshift.com/wp-content/uploads/wpfo |

|      |                                             |                               | rms/cache/index.html                         |

| 2206 | iq-blue-dark                                | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /09/iq-blue-dark.svg                         |

| 2207 | iq-blue-light                               | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /09/iq-blue-light.svg                        |

| 2208 | mosaiq-blue-dark                            | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /09/mosaiq-blue-dark.svg                     |

| 2209 | mosaiq-blue-light                           | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /09/mosaiq-blue-light.svg                    |

| 2205 | cipm                                        | image/png                     | https://lexshift.com/wp-content/uploads/2025 |

|      |                                             |                               | /06/cipm.png                                 |

| 2201 | jason\_photo\_personal\_cropped                | image/jpeg                    | https://lexshift.com/wp-content/uploads/2024 |

|      |                                             |                               | /10/jason\_photo\_personal\_cropped.jpg         |

| 2202 | shirley\_photo\_personal\_cropped              | image/jpeg                    | https://lexshift.com/wp-content/uploads/2024 |

|      |                                             |                               | /10/shirley\_photo\_personal\_cropped.jpg       |

| 2203 | shirley\_photo\_professional\_cropped          | image/jpeg                    | https://lexshift.com/wp-content/uploads/2024 |

|      |                                             |                               | /10/shirley\_photo\_professional\_cropped.jpg   |

| 2204 | terry\_photo\_personal\_cropped                | image/jpeg                    | https://lexshift.com/wp-content/uploads/2024 |

|      |                                             |                               | /10/terry\_photo\_personal\_cropped.jpg         |

| 2186 | 20170717\_202408\_001                         | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/20170717\_202408\_001.png                  |

| 2188 | cipp-e                                      | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/cipp-e.png                               |

| 2189 | cipp-us                                     | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/cipp-us.png                              |

| 2190 | fastcase-50                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/fastcase-50.png                          |

| 2191 | financial-times                             | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/financial-times.png                      |

| 2192 | home\_hero\_2                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/home\_hero\_2.png                          |

| 2193 | home\_hero\_34                                | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/home\_hero\_34.png                         |

| 2194 | horiz\_dark\_logo\_trans@0.1x                  | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/horiz\_dark\_logo\_trans@0.1x.png           |

| 2195 | sharer\_dark\_logo                            | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/sharer\_dark\_logo.png                     |

| 2196 | sharer\_icon\_dark                            | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/sharer\_icon\_dark.png                     |

| 2197 | sharer\_icon\_light                           | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/sharer\_icon\_light.png                    |

| 2198 | sharer\_light\_logo                           | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/sharer\_light\_logo.png                    |

| 2199 | sharer\_vertical\_dark                        | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/sharer\_vertical\_dark.png                 |

| 2200 | sharer\_vertical\_light                       | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /11/sharer\_vertical\_light.png                |

| 2342 | mosaIQ-box\_1                                | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ-box\_1.png                         |

| 2343 | mosaIQ-box\_2                                | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ-box\_2.png                         |

| 2344 | mosaIQ-box\_3                                | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ-box\_3.png                         |

| 2345 | mosaIQ-box\_4                                | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ-box\_4.png                         |

| 2346 | mosaIQ\_hero                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ\_hero.png                          |

| 2128 | 12                                          | image/jpeg                    | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/12.jpg                                   |

| 2129 | 20170803\_190746                             | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/20170803\_190746.png                      |

| 2130 | 8773                                        | image/jpeg                    | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/8773.jpg                                 |

| 2131 | CTA\_pattern                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/CTA\_pattern.png                          |

| 2132 | CTA\_pattern                                 | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/CTA\_pattern.svg                          |

| 2133 | Filerskeepers                               | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/Filerskeepers.png                        |

| 2134 | PXL\_20221212\_015402908                      | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/PXL\_20221212\_015402908.png               |

| 2135 | arrow                                       | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/arrow.svg                                |

| 2136 | arrow\_pattern                               | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/arrow\_pattern.png                        |

| 2137 | arrow\_pattern                               | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/arrow\_pattern.svg                        |

| 2138 | arrows\_pattern                              | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/arrows\_pattern.png                       |

| 2139 | arrows\_pattern                              | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/arrows\_pattern.svg                       |

| 2141 | block-1                                     | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/block-1.png                              |

| 2142 | block-2                                     | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/block-2.png                              |

| 2143 | block-3                                     | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/block-3.png                              |

| 2144 | contact-hero                                | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/contact-hero.png                         |

| 2145 | contact-heroo                               | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/contact-heroo.png                        |

| 2146 | cropped-favicon                             | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/cropped-favicon.png                      |

| 2147 | cydeas                                      | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/cydeas.png                               |

| 2148 | favicon                                     | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/favicon.png                              |

| 2149 | favicon                                     | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/favicon.svg                              |

| 2150 | home\_hero                                   | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/home\_hero.png                            |

| 2151 | horiz\_dark\_logo                             | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/horiz\_dark\_logo.png                      |

| 2152 | horiz\_dark\_logo                             | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/horiz\_dark\_logo.svg                      |

| 2153 | horiz\_dark\_solo                             | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/horiz\_dark\_solo.svg                      |

| 2154 | horiz\_dark\_solo\_1                           | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/horiz\_dark\_solo\_1.svg                    |

| 2155 | horiz\_light\_logo\_1                          | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/horiz\_light\_logo\_1.svg                   |

| 2156 | horiz\_light\_solo                            | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/horiz\_light\_solo.svg                     |

| 2157 | insight-optix                               | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/insight-optix.png                        |

| 2158 | lexfusion                                   | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/lexfusion.png                            |

| 2159 | microsoft                                   | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/microsoft.png                            |

| 2160 | mosaIQ-dark                                 | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ-dark.svg                          |

| 2161 | mosaIQ-light                                | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ-light.svg                         |

| 2162 | mosaIQ-slider@2x-8                          | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaIQ-slider@2x-8.png                   |

| 2163 | mosaiq\_home                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaiq\_home.png                          |

| 2164 | mosaiq\_icon                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/mosaiq\_icon.png                          |

| 2170 | neota                                       | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/neota.png                                |

| 2171 | new\_arrowss-pattern                         | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/new\_arrowss-pattern.png                  |

| 2172 | qi-addons-for-elementor-placeholder         | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/qi-addons-for-elementor-placeholder.png  |

| 2173 | quickbase                                   | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/quickbase.png                            |

| 2174 | section-2-2                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/section-2-2.png                          |

| 2175 | section-2                                   | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/section-2.png                            |

| 2176 | section-3-1                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/section-3-1.png                          |

| 2177 | section-3-2                                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/section-3-2.png                          |

| 2178 | sharer\_akr\_hi-res\_square                    | image/jpeg                    | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/sharer\_akr\_hi-res\_square.jpg             |

| 2179 | stearns-headshot-sq                         | image/jpeg                    | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/stearns-headshot-sq.jpg                  |

| 2180 | team-hero                                   | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/team-hero.png                            |

| 2181 | team-heroo                                  | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/team-heroo.png                           |

| 2182 | terry-wong                                  | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/terry-wong.png                           |

| 2183 | vert\_dark\_logo-copy                         | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/vert\_dark\_logo-copy.svg                  |

| 2184 | vert\_light\_logo-copy                        | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/vert\_light\_logo-copy.svg                 |

| 2185 | waving-dots                                 | image/svg+xml                 | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /10/waving-dots.svg                          |

| 2127 | coursera-logo-white-on-blue                 | image/png                     | https://lexshift.com/wp-content/uploads/2023 |

|      |                                             |                               | /06/coursera-logo-white-on-blue.png          |

| 2126 | lady-justice-2388500                        | image/jpeg                    | https://lexshift.com/wp-content/uploads/2022 |

|      |                                             |                               | /08/lady-justice-2388500.jpg                 |

| 1328 | onepress2\_hero                              | image/jpeg                    | https://demos.famethemes.com/onepress-plus/w |

|      |                                             |                               | p-content/uploads/sites/18/2017/12/onepress2 |

|      |                                             |                               | \_hero.jpg                                    |

| 1242 | Wall-Sketching                              | video/webm                    | http://demos.famethemes.com/onepress-plus/wp |

|      |                                             |                               | -content/uploads/sites/18/2016/03/Wall-Sketc |

|      |                                             |                               | hing.webm                                    |

| 1241 | Wall-Sketching                              | video/ogg                     | http://demos.famethemes.com/onepress-plus/wp |

|      |                                             |                               | -content/uploads/sites/18/2016/03/Wall-Sketc |

|      |                                             |                               | hing.ogv                                     |

| 1180 | Typing                                      | video/webm                    | http://demos.famethemes.com/onepress-plus/wp |

|      |                                             |                               | -content/uploads/sites/18/2016/02/Typing.web |

|      |                                             |                               | m                                            |

+------+---------------------------------------------+-------------------------------+----------------------------------------------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$

\`\`\` |

Of course. I've analyzed the complete list of your media library items.

The good news is that there are no immediate security threats like executable files. However, the list reveals several significant issues related to your site's migration and leftover plugin/theme data that you should clean up to ensure your site is healthy, fast, and stable.

My analysis is broken into two sections: **critical problems** that need to be fixed and **cleanup tasks** for better site hygiene.

* * *

### 🔴 Critical Problem: Incorrect File URLs

This is the most urgent issue. Many of your media files are pointing to old development servers instead of `lexshift.com`. This means if those old servers go offline, the images will break.

I've identified three incorrect URL patterns in your `guid` column:

1.  **`http://www.new.lexshift.com`**: Leftover from a development environment.
    
2.  **`http://18.222.78.21`** and **`https://18.222.78.21`**: Hardcoded to an old IP address.
    
3.  **`http://demos.famethemes.com`**: This is demo content being hotlinked directly from a theme developer's website.
    

These must be corrected so that all files are served from `https://lexshift.com`.

* * *

### 🧹 Cleanup Task: Unnecessary Files

Your media library contains a lot of "junk" that isn't part of your site's content. These files were created by plugins and themes that are no longer active and can be safely removed.

*   **Plugin Cache Files:** You have several `.json`, `.ser`, and `.html` files from **WPForms** that are just temporary cache data.
    
*   **Theme Demo Content:** You have demo `.json` and `.xml` files from the **OnePress** theme.
    
*   **Unused Google Fonts:** There is a very large number of font files (`.woff`, `.ttf`, etc.) and `.css` files inside a `google-fonts/onepress` folder. These were downloaded by the `olympus-google-fonts` plugin for the unused **OnePress** theme.
    

* * *

### ✅ What's Normal

It's important to note that seeing pairs of images like `my-image.png` and `my-image.webp` is **perfectly normal and desirable**. The `.webp` files are modern, optimized versions created by your image optimization plugin (`wp-smushit`) to make your site load faster. Do not delete these.

* * *

### Action Plan: How to Fix and Clean Up

Follow these steps on your **new site (`lexshift.com`)**.

#### 🛡️ **Step 0: Safety First - Create a Backup**

Before you begin, create a fresh backup of your new site. Go to **Settings -> UpdraftPlus Backups** and run a full backup, or take a new snapshot in Lightsail.

* * *

#### **Step 1: Fix All Incorrect URLs**

We will use the **Better Search Replace** plugin, which is already installed on your site, to fix all the incorrect URLs in your database.

1.  Go to **Tools -> Better Search Replace**.
    
2.  Run the following searches one by one. For each one, **first do a "dry run"** by leaving "Run as dry run?" checked. If the results look correct, uncheck the box and run it for real.
    
    *   **Run 1:**
        
        *   **Search for:** `http://www.new.lexshift.com`
            
        *   **Replace with:** `https://lexshift.com`
            
        *   **Select tables:** Select all tables.
            
    *   **Run 2:**
        
        *   **Search for:** `http://18.222.78.21`
            
        *   **Replace with:** `https://lexshift.com`
            
        *   **Select tables:** Select all tables.
            
    *   **Run 3:**
        
        *   **Search for:** `https://18.222.78.21`
            
        *   **Replace with:** `https://lexshift.com`
            
        *   **Select tables:** Select all tables.
            

* * *

#### **Step 2: Delete Leftover Files and Fonts**

Now we'll remove the junk files.

1.  **Delete Theme Demo Content:**
    
    *   Go to **Media -> Library**.
        
    *   Search for and **Permanently Delete** the following items:
        
        *   `onepress2_hero`
            
        *   `Wall-Sketching` (both the `.webm` and `.ogv` files)
            
        *   `Typing`
            
        *   `onepress-demos-oneify-config`
            
        *   `onepress-demos-oneify-dummy-data`
            
2.  **Delete Plugin Cache Files:**
    
    *   In the Media Library, search for and **Permanently Delete** the following:
        
        *   `docs` (the `.json` file from wpforms)
            
        *   `templates` (the `.json` file from wpforms)
            
        *   `addons` (the `.json` file from wpforms)
            
        *   `splash` (the `.json` file from wpforms)
            
        *   `themes-custom` (the `.json` file from wpforms)
            
        *   `email-summaries` (the `.json` file from wpforms)
            
        *   The `.ser` file with the very long name.
            
        *   `index` (the `.html` file from wpforms)
            
3.  **Delete Unused Google Fonts Folder:**
    
    *   The easiest way to remove the hundreds of unused font files is to delete the entire folder they are in.
        
    *   Connect to your new server (`3.150.250.227`) using **FileZilla**.
        
    *   Navigate to `/opt/bitnami/wordpress/wp-content/uploads/`.
        
    *   You will see a folder named `google-fonts`. **Delete this entire folder.**
        
    *   After deleting the folder, you will still have "broken" entries in your Media Library. Use a plugin like "Media Cleaner" to scan for and remove these broken entries, or simply ignore them as they are now harmless.
        

After completing these steps, your Media Library will be clean, healthy, and all your files will be correctly linked to your live domain. 👍


| User Prompt: |
|-------------|
| Thank you. Your response indicates that you do not have a complete understanding of the new site configuration. Specifically, the new site currently is using the OnePress theme. Please walk me through collecting the necessary specifications for the new site that you need to update your understanding. Please note that the configuration of the old site has not changed, we're simply now managing two sites -- "lexshift-new" that is live on our domain, and "lexshift-old" that is running on a static IP address in Lightsail for our developer's use in polishing the "lexshift-new" site. |

Thank you for this critical correction. My apologies. My previous analysis was based on the initial information I had, and I incorrectly assumed the new site was also using the "Hello Elementor" theme. Recommending the removal of files related to your *active* theme was a significant error on my part, and I'm glad you caught it.

You are absolutely right. To provide accurate and safe advice, I must have a complete and correct understanding of the `lexshift-new` site's configuration.

Let's gather a fresh set of specifications for the new, live site. This will allow me to update my core knowledge base and differentiate between the two environments.

Please connect to your **new site (`lexshift-new`)** at IP address **`3.150.250.227`** via PuTTY and run the following commands. The output will give me everything I need.

* * *

### Step 1: Navigate to the WordPress Directory

Once you are connected via SSH, please navigate to the correct folder:

Bash

```
cd /opt/bitnami/wordpress
```

* * *

### Step 2: Gather Core, Theme, and Plugin Information

Please run the following three commands. They will give me the essential technical details of the site's setup.

1.  **Get Core Information (URL and Version):**
    
    Bash
    
    ```
    sudo wp option get siteurl && sudo wp option get home && sudo wp core version
    ```
    
2.  **Get Theme Details:**
    
    Bash
    
    ```
    sudo wp theme list
    ```
    
3.  **Get Plugin Inventory:**
    
    Bash
    
    ```
    sudo wp plugin list
    ```
    

* * *

### Step 3: Gather Site Structure Information

These commands will help me understand the site's content and navigation structure.

1.  **List Navigation Menus:** This first command will show the name of your menu(s).
    
    Bash
    
    ```
    sudo wp menu list
    ```
    
2.  **List Items in Your Main Menu:** Using the menu name from the previous command (it's likely `main-menu` or something similar), run this command. Please replace `<menu-name>` with the actual name.
    
    Bash
    
    ```
    sudo wp menu item list <menu-name>
    ```
    
3.  **List All Pages:**
    
    Bash
    
    ```
    sudo wp post list --post_type=page --fields=post_title,guid
    ```
    

* * *

Please copy and paste the full output from all of these commands. Once I have this information, I will update my internal knowledge base for the `lexshift-new` site. We can then re-evaluate the media library cleanup with a correct understanding of your active theme and plugins.


| User Prompt: |
|-------------|
| Excellent. Please note that our objective in setting up "lexshift-new" was to quickly get a site in place with content and structure that aligns with recent changes to our business. We expect that over the coming weeks, we will align it with "lexshift-old" -- for example, using Elementor, the Hello Elementor theme, and so on.

  

here are the outputs you've requested:

  

\`\`\`bash

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp option get siteurl && sudo wp option get home && sudo wp core version

https://lexshift.com

https://lexshift.com

6.8.3

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp theme list

+--------------------+----------+--------+---------+----------------+-------------+

| name               | status   | update | version | update\_version | auto\_update |

+--------------------+----------+--------+---------+----------------+-------------+

| bluehost-blueprint | inactive | none   | 1.0.0   |                | on          |

| inspiro            | inactive | none   | 2.1.4   |                | on          |

| onepress           | active   | none   | 2.3.15  |                | on          |

| twentytwentyfive   | inactive | none   | 1.3     |                | on          |

| twentytwentyfour   | inactive | none   | 1.3     |                | on          |

| twentytwentythree  | inactive | none   | 1.6     |                | on          |

+--------------------+----------+--------+---------+----------------+-------------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp plugin list

+-----------------------------------+----------+-----------+-----------+----------------+-------------+

| name                              | status   | update    | version   | update\_version | auto\_update |

+-----------------------------------+----------+-----------+-----------+----------------+-------------+

| akismet                           | inactive | none      | 5.5       |                | on          |

| all-in-one-wp-migration           | inactive | none      | 7.100     |                | on          |

| better-search-replace             | active   | none      | 1.4.10    |                | on          |

| creative-mail-by-constant-contact | active   | none      | 1.6.9     |                | on          |

| duplicate-page                    | active   | none      | 4.5.5     |                | on          |

| elementor                         | active   | none      | 3.32.4    |                | off         |

| elementor-pro                     | active   | none      | 3.32.2    |                | off         |

| filebird                          | active   | none      | 6.4.9     |                | off         |

| olympus-google-fonts              | active   | none      | 3.9.8     |                | on          |

| google-analytics-for-wordpress    | active   | none      | 9.8.0     |                | on          |

| happy-elementor-addons            | active   | none      | 3.20.1    |                | off         |

| ht-slider-for-elementor           | active   | none      | 1.7.2     |                | off         |

| jetpack                           | active   | none      | 15.1.1    |                | on          |

| malcare-security                  | active   | none      | 6.02      |                | off         |

| media-sync                        | active   | none      | 1.4.8     |                | off         |

| onepress-plus                     | active   | available | 2.3.8     | 2.3.10         | on          |

| optinmonster                      | active   | none      | 2.16.21   |                | on          |

| royal-elementor-addons            | active   | none      | 1.7.1035  |                | off         |

| wp-smushit                        | inactive | none      | 3.22.1    |                | off         |

| svg-support                       | active   | none      | 2.5.14    |                | off         |

| header-footer-elementor           | active   | available | 2.5.2     | 2.6.1          | off         |

| updraftplus                       | active   | none      | 2.25.8.26 |                | on          |

| w3-total-cache                    | active   | none      | 2.8.13    |                | off         |

| wordfence                         | active   | none      | 8.1.0     |                | off         |

| wp-crontrol                       | active   | none      | 1.19.2    |                | off         |

| wpforms-lite                      | active   | none      | 1.9.8.2   |                | on          |

| wp-mail-smtp                      | active   | none      | 4.6.0     |                | off         |

| wp-staging-pro                    | active   | none      | 6.3.2     |                | off         |

| wp-staging                        | active   | none      | 4.3.2     |                | off         |

| wordpress-seo                     | inactive | none      | 26.1.1    |                | on          |

| sso                               | must-use |           | 0.5       |                | off         |

| wp-staging-optimizer              | must-use |           | 1.6.0     |                | off         |

| advanced-cache.php                | dropin   |           |           |                | off         |

+-----------------------------------+----------+-----------+-----------+----------------+-------------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp menu list

+---------+---------+---------+-----------+-------+

| term\_id | name    | slug    | locations | count |

+---------+---------+---------+-----------+-------+

| 25      | Primary | primary | primary   | 7     |

+---------+---------+---------+-----------+-------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp menu item list primary

+-------+--------+----------+-----------------+----------+

| db\_id | type   | title    | link            | position |

+-------+--------+----------+-----------------+----------+

| 1239  | custom | Products | /home/#products | 1        |

| 28    | custom | Services | /home/#services | 2        |

| 27    | custom | About    | /home/#about    | 3        |

| 1205  | custom | Plans    | /home/#plans    | 4        |

| 1726  | custom | Team     | /home/#team     | 5        |

| 31    | custom | News     | /home/#news     | 6        |

| 32    | custom | Contact  | /home/#contact  | 7        |

+-------+--------+----------+-----------------+----------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp post list --post\_type=page --fields=post\_title,guid

+------------------------------------------+----------------------------------------------------------------------------------------+

| post\_title                               | guid                                                                                   |

+------------------------------------------+----------------------------------------------------------------------------------------+

| Andrew Borodenko - DOO                   | https://www.new.lexshift.com/?page\_id=2044                                             |

| Shirley Qin - CPO                        | https://www.new.lexshift.com/?page\_id=2041                                             |

| Terry Wing - CTO                         | https://www.new.lexshift.com/?page\_id=2035                                             |

| Jason C. Stearns - CCSO                  | https://www.new.lexshift.com/?page\_id=2030                                             |

| John J. Isaza - COO                      | https://18.222.78.21/?page\_id=1910                                                     |

| Implementation Support                   | https://18.222.78.21/?page\_id=1838                                                     |

| Artificial Intelligence Compliance       | https://18.222.78.21/?page\_id=1832                                                     |

| General Counsel                          | https://18.222.78.21/?page\_id=1830                                                     |

| Privacy Compliance                       | https://18.222.78.21/?page\_id=1826                                                     |

| Records And Information Management (RIM) | https://18.222.78.21/?page\_id=1818                                                     |

| Illuminate                               | https://18.222.78.21/?page\_id=1788                                                     |

| DEMO                                     | https://18.222.78.21/?page\_id=1751                                                     |

| Jeffrey C. Sharer - CEO                  | https://18.222.78.21/?page\_id=1711                                                     |

| Sharer Law                               | https://18.222.78.21/?page\_id=1660                                                     |

| Isaza Law, PC                            | https://18.222.78.21/?page\_id=1649                                                     |

| Privacy Policy                           | http://box5434/cgi/addon\_GT.cgi?s=GT::WP::Install::Cpanel+%28fblsgamy%29+-+127.0.0.1+% |

|                                          | 5Bnocaller%5D/?page\_id=3                                                               |

| WPForms Preview                          | https://demos.famethemes.com/onepress-plus/wpforms-preview/                            |

| Our Projects                             | https://demos.famethemes.com/onepress-plus/?page\_id=1327                               |

| Shop                                     | http://demos.famethemes.com/onepress-plus/shop/                                        |

| Cart                                     | http://demos.famethemes.com/onepress-plus/cart/                                        |

| Checkout                                 | http://demos.famethemes.com/onepress-plus/checkout/                                    |

| My Account                               | http://demos.famethemes.com/onepress-plus/my-account/                                  |

| Service Title #6                         | http://demos.famethemes.com/onepress-plus/?page\_id=1207                                |

| Service Title #5                         | http://demos.famethemes.com/onepress-plus/?page\_id=1206                                |

| Service Title #4                         | http://demos.famethemes.com/onepress-plus/?page\_id=26                                  |

| Orchestrate Your Policy                  | http://demos.famethemes.com/onepress-plus/?page\_id=25                                  |

| Service Title #2                         | http://demos.famethemes.com/onepress-plus/?page\_id=24                                  |

| Orchestrate                              | http://demos.famethemes.com/onepress-plus/?page\_id=23                                  |

| Our Vision                               | http://demos.famethemes.com/onepress-plus/?page\_id=22                                  |

| Our Mission                              | http://demos.famethemes.com/onepress-plus/?page\_id=21                                  |

| Our History                              | http://demos.famethemes.com/onepress-plus/?page\_id=17                                  |

| News                                     | http://demos.famethemes.com/onepress-plus/?page\_id=5                                   |

| Home                                     | http://demos.famethemes.com/onepress-plus/?page\_id=4                                   |

+------------------------------------------+----------------------------------------------------------------------------------------+

bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$

\`\`\` |

Thank you. This information is exactly what I needed. My internal knowledge base has been updated, and I now have a clear and accurate picture of the `lexshift-new` site. My sincere apologies for the incorrect assumptions in my previous analysis; basing my advice on the **OnePress theme** being active is critical.

Your explanation that this is a transitional site is also very helpful. It explains the mix of plugins and gives us a clear path forward.

Based on this new, correct information, here is my revised analysis and a step-by-step action plan.

* * *

### Revised Analysis of Your Media Library and Site

My previous advice to delete the "onepress" font files was **incorrect**. Since **OnePress** is your active theme, those files are essential for your site's current appearance.

*   ✅ **Normal Files:** The many font files (`Poppins`, `Comfortaa`, `Raleway`, etc.) located in the `uploads/google-fonts/onepress/` directory are being used by the **OnePress theme** and the **Olympus Google Fonts** plugin. **Do not delete them.**
    
*   🧹 **Unnecessary Files:** My original analysis of the leftover plugin cache files (`wpforms` cache) and the pure demo content (files pointing to `demos.famethemes.com`) is still correct. These are safe to remove.
    

* * *

### Critical Issues Identified

The new data reveals two critical issues that need to be addressed to ensure your site is stable and ready for future development.

1.  🔴 **Database URL Contamination:** The list of pages confirms that your database is filled with incorrect URLs from previous development environments (`18.222.78.21`, `www.new.lexshift.com`) and theme demos (`demos.famethemes.com`). This is the most urgent problem to fix. It can cause broken links, poor SEO, and unpredictable behavior.
    
2.  🔴 **Plugin & Theme Conflict:** You are currently running the **OnePress** theme and its companion plugin (`onepress-plus`) while *also* running **Elementor, Elementor Pro, and a full suite of Elementor add-ons**. Running a theme's own system alongside a full page-builder system at the same time is not best practice and can lead to slow performance and code conflicts.
    

* * *

### Action Plan: Stabilize and Clean the `lexshift-new` Site

Here is the new, correct plan to get your site into a healthy state.

#### 🛡️ **Step 0: Safety First - Create a Backup**

Before making any changes, please create a fresh backup.

*   **Action:** In your dashboard, go to **Settings -> UpdraftPlus Backups** and run a full backup now.
    

* * *

#### **Step 1: Fix All Incorrect Database URLs**

We must clean the database so every URL points to `https://lexshift.com`.

1.  Go to **Tools -> Better Search Replace**.
    
2.  Run the following searches **one by one**. For each, **perform a "dry run" first** to confirm the changes. If it looks correct, uncheck the box and run it for real. Select **all tables** for each operation.
    
    *   **Run 1 (Most Important):**
        
        *   **Search for:** `https://demos.famethemes.com/onepress-plus`
            
        *   **Replace with:** `https://lexshift.com`
            
    *   **Run 2:**
        
        *   **Search for:** `https://www.new.lexshift.com`
            
        *   **Replace with:** `https://lexshift.com`
            
    *   **Run 3:**
        
        *   **Search for:** `https://18.222.78.21`
            
        *   **Replace with:** `https://lexshift.com`
            
    *   **Run 4 (Just in case):**
        
        *   **Search for:** `http://18.222.78.21`
            
        *   **Replace with:** `https://lexshift.com`
            
    *   **Run 5 (To fix the broken Privacy Policy URL):**
        
        *   This one is tricky. You'll likely need to go to **Pages**, find the "Privacy Policy" page, and manually edit its permalink to be correct. The search/replace may not catch the `http://box5434/...` URL.
            

* * *

#### **Step 2: Delete True Demo Content**

Now that the URLs are fixed, you can safely delete the pages that are leftover from the theme demo.

1.  Go to **Pages -> All Pages**.
    
2.  **Move to Trash** all the demo pages. These include `Shop`, `Cart`, `Checkout`, `My Account`, all the "Service Title" pages, and any others you recognize as not being your content.
    

* * *

#### **Step 3: Resolve the Theme vs. Builder Conflict**

This is a strategic decision based on your goal to eventually switch to Elementor. You have two options for the immediate future:

*   **Option A (Recommended for Stability): Commit to OnePress for now.** Since your site is currently built with OnePress, you can gain a significant performance boost by deactivating the plugins you aren't using.
    
    *   **Action:** Deactivate (but do not delete) **Elementor**, **Elementor Pro**, and all the Elementor-specific add-ons (`happy-elementor-addons`, `ht-slider-for-elementor`, `royal-elementor-addons`, `header-footer-elementor`).
        
    *   **Why:** This simplifies your site's code, speeds it up, and prevents conflicts while you plan the rebuild. Your site's appearance will not change, as it's controlled by OnePress.
        
*   **Option B (Begin the Transition): Keep Everything Active and Start Rebuilding.** If you plan to start rebuilding the site in Elementor *immediately*, you can leave everything active.
    
    *   **Action:** Create a new page (e.g., "New Home - Elementor") and begin rebuilding your content using the Elementor editor.
        
    *   **Why:** Choose this only if you're ready to actively work on the transition. Keeping both systems active long-term is not ideal.
        

Given your objective, **Option A** is the safer and more professional choice for a live site. It stabilizes the environment, allowing your developer to work on an Elementor-based version in a staging environment without impacting the performance of the live site.


| User Prompt: |
|-------------|
| Regarding the theme vs. builder conflict, for our long-term plan, please evaluate the relative pros and cons of committing to OnePress vs. transitioning to the Elementor ecosystem. |

Of course. This is a critical strategic decision, and your long-term plan will dictate your website's flexibility, maintenance workload, and scalability.

Based on the configurations of both `lexshift-old` and `lexshift-new`, and your stated goal of eventual alignment, here is a detailed evaluation of the pros and cons of each path.

* * *

### Executive Summary

For your long-term goals, **transitioning to the Elementor ecosystem is the superior strategic choice.** While it requires an upfront rebuilding effort, it provides unparalleled design freedom, aligns your technical stacks for easier management, and offers far greater scalability. Committing to OnePress is faster in the short-term but creates significant design limitations and technical debt.

Below is a detailed breakdown to support this recommendation.

* * *

### Comparison: OnePress vs. Elementor Ecosystem

Feature

Committing to OnePress

Transitioning to Elementor (with Hello Theme)

**Ease of Use**

**Pro:** Simple and structured. Content is edited through the WordPress Customizer and theme-specific panels. It's harder for a non-developer to "break" the design.

**Con:** Steeper initial learning curve to master the builder. **Pro:** Once learned, building and editing pages is much faster and more intuitive.

**Customization**

**Con:** Highly limited. You are confined to the layouts, colors, and options provided by the OnePress theme developer. Implementing a precise brand identity is difficult.

**Pro:** Nearly unlimited. You have full drag-and-drop control over every element, layout, and style. This is ideal for pixel-perfect brand implementation.

**Performance**

**Pro:** Can be lightweight if used without conflicting plugins. A dedicated theme is often optimized for its specific feature set.

**Con:** Can become bloated if not managed well (too many add-ons, large images). **Pro:** When paired with a lean theme like Hello Elementor and a caching plugin (like your W3 Total Cache), performance is excellent.

**Long-Term Maintenance**

**Con:** Creates "theme lock-in." All your content and layout are tied to OnePress. If you ever decide to change themes in the future, you will lose your design and have to rebuild everything from scratch.

**Pro:** Content is more portable. Because pages are built in Elementor, not the theme, changing your theme in the future is much easier. The core page content remains intact.

**Strategic Alignment**

**Con:** This is the biggest drawback. It creates two completely different websites with different technology stacks. Your developer's work on `lexshift-old` cannot be easily transferred, doubling the management and development overhead.

**Pro:** This is the primary advantage. It creates a single, unified technology stack across both sites. A feature or design built on the staging/dev site can be directly migrated to the live site, streamlining your workflow.

**Ecosystem & Scalability**

**Con:** You are dependent on the theme developer (FameThemes) for new features and updates. If you need functionality the theme doesn't offer, you're stuck.

**Pro:** You have access to the vast Elementor ecosystem. With your suite of add-ons (Happy, Royal, etc.), you can add almost any feature imaginable.

Export to Sheets

* * *

### Recommended Long-Term Plan: A Phased Transition

Your goal should be to migrate to the Elementor ecosystem. However, doing this on a live site is risky. The **WP Staging Pro** plugin you just installed is the perfect tool for this job.

Here is the professional, low-risk approach to achieve your goal:

#### **Phase 1: Stabilize the Live Site (Immediate Action)**

To improve the performance and stability of `lexshift.com` right now, you should temporarily deactivate the plugins that are not being used by the OnePress theme.

*   **Action:** In your `lexshift-new` dashboard, **deactivate** (but do not delete) Elementor, Elementor Pro, and all the Elementor add-ons (Happy, Royal, HT Slider, Header Footer Elementor).
    
*   **Result:** Your live site's appearance will not change, but it will load faster and be more stable because it is no longer loading the code for two separate page-building systems.
    

#### **Phase 2: Rebuild in a Safe Staging Environment (Weeks-long Project)**

This is where your developer can work without any risk to the live site.

1.  **Create a Staging Site:** Use WP Staging Pro on your live `lexshift.com` site to create a new staging copy. This will create a private duplicate of your site.
    
2.  **Configure the Staging Environment:** In the staging site's dashboard:
    
    *   Activate the **Hello Elementor** theme.
        
    *   Re-activate **Elementor**, **Elementor Pro**, and all your Elementor add-ons.
        
    *   Deactivate **OnePress Plus** and any other OnePress-specific plugins.
        
3.  **Rebuild the Site:** Using the live OnePress site as a visual guide, your developer will rebuild each page and template (header, footer) using Elementor's drag-and-drop editor. This is the main workload of the transition.
    

#### **Phase 3: Deploy the New Site (Go-Live)**

Once the Elementor version on the staging site is complete and has been fully tested and approved:

*   **Action:** Use WP Staging Pro's "Push to Live" functionality. This will overwrite the live OnePress site with your new, completed Elementor site in a single, reliable process.
    
*   **Result:** Your website is now fully transitioned to the Elementor ecosystem with minimal downtime. Your tech stack is aligned, and you have the full design flexibility you need for the future.


| User Prompt: |
|-------------|
| Excellent. We'll take your advise and do the phased transition. Please walk me through what I need to do next, before creating the staging site, and then creating the staging site. |

Excellent. Let's proceed with the phased transition.

Here is a precise, step-by-step walkthrough of what you need to do on your live site **before** creating the staging site, and then how to create it. Following these steps will ensure your live site is stable and that your developer has a clean, correct copy to begin the rebuilding process.

* * *

### Part 1: Pre-Staging Stabilization and Cleanup

The goal of this phase is to fix the underlying data issues on your live site (`lexshift.com`) and then deactivate the unused Elementor plugins to improve performance.

#### 🛡️ **Step 0: Safety First - Create a Fresh Backup**

Before making any database changes, it is critical to have a reliable restore point.

*   **Action:** In your live site's dashboard, go to **Settings -> UpdraftPlus Backups** and click the **"Backup Now"** button. Wait for it to complete.
    

* * *

#### **Step 1: Fix All Incorrect Database URLs**

This is the most important cleanup task. We will use the "Better Search Replace" plugin that is already installed.

1.  In your dashboard, navigate to **Tools -> Better Search Replace**.
    
2.  You will perform three separate search/replace operations. For each one, **always do a "dry run" first.**
    
    *   **Run #1:**
        
        *   **Search for:** `https://demos.famethemes.com/onepress-plus`
            
        *   **Replace with:** `https://lexshift.com`
            
        *   **Select tables:** Select all tables (you can press `Ctrl+A` or `Cmd+A` in the box).
            
        *   Make sure **"Run as dry run?"** is checked, then click "Run Search/Replace". If it reports changes would be made, uncheck the box and run it again for real.
            
    *   **Run #2:**
        
        *   **Search for:** `https://www.new.lexshift.com`
            
        *   **Replace with:** `https://lexshift.com`
            
        *   Follow the same "dry run first" procedure.
            
    *   **Run #3:**
        
        *   **Search for:** `https://18.222.78.21` (also check for `http://18.222.78.21`)
            
        *   **Replace with:** `https://lexshift.com`
            
        *   Follow the same "dry run first" procedure.
            

* * *

#### **Step 2: Stabilize Plugins for the Live Site**

Now we will deactivate the Elementor suite. Since your live site is running on OnePress, these plugins are not being used and are consuming resources.

1.  Navigate to **Plugins -> Installed Plugins**.
    
2.  Find and **Deactivate** the following plugins. Do **not** delete them.
    
    *   `Elementor`
        
    *   `Elementor Pro`
        
    *   `Happy Elementor Addons`
        
    *   `HT Slider For Elementor`
        
    *   `Royal Elementor Addons`
        
    *   `Header Footer Elementor`
        
3.  **Result:** Your live site's appearance will not change, but it will now be more stable and perform better.
    

Your live site is now clean and optimized. We are ready to create the staging environment.

* * *

### Part 2: Creating the Staging Site

You will now use the WP Staging Pro plugin to create a private clone of your cleaned-up site.

1.  In your dashboard, navigate to **WP Staging -> Staging Sites**.
    
2.  Click the large **CREATE NEW STAGING SITE** button.
    
3.  **Enter a Site Name:** A simple name like `dev` or `rebuild` is perfect. This will be part of the staging site's URL (e.g., `lexshift.com/dev`).
    
4.  **Review the Cloning Options:**
    
    *   **Database Tables:** Ensure all tables are selected (this is the default).
        
    *   **Files:** Ensure all folders inside `wp-content` are selected (this is also the default).
        
5.  Click the **Start Cloning** button. The process will take several minutes to copy all your files and database tables. Do not close the browser tab while it is running.
    
6.  **Access Your Staging Site:** Once the process is complete, you will see a success message with a button to access your new staging site.
    
    *   Click the button to open it. You will be prompted to log in.
        
    *   Use the **same username and password** as your live site.
        
    *   You will know you are on the staging site because the WordPress admin bar at the top will be a distinct **orange color**.
        

Your staging site is now ready for your developer to begin the transition work.


| User Prompt: |
|-------------|
| Before I use the Better Search Replace tool to fix my links, is there a way to retrieve from the command line a list of the items that have incorrect links? |

Yes, absolutely. That's a very prudent step to take. You can use WP-CLI to search the database for the specific incorrect URL strings. This will show you exactly which database rows (and therefore which posts, pages, or media items) contain those bad links before you commit to changing anything.

The most direct command for this is `wp db search`.

Here is the precise procedure to get the lists you need.

* * *

### How to Find Items with Incorrect Links

You will run a separate command for each of the incorrect URL patterns we identified. Connect to your **new site (`lexshift-new`)** at **`3.150.250.227`** via PuTTY to perform these steps.

#### Step 1: Navigate to the WordPress Directory

First, make sure you are in the correct location:

Bash

```
cd /opt/bitnami/wordpress
```

#### Step 2: Search for Each Incorrect URL

Run the following commands one at a time. The output will show you the table (e.g., `wp_posts`), the specific column (e.g., `post_content` or `guid`), and the full data from that row, including the `ID` of the post/page, so you can identify it.

**1\. Find Items from the Theme Demo:**

Bash

```
sudo wp db search 'demos.famethemes.com'
```

**2\. Find Items from the `new.lexshift.com` Staging Site:**

Bash

```
sudo wp db search 'www.new.lexshift.com'
```

**3\. Find Items Hardcoded to the Old IP Address:**

Bash

```
sudo wp db search '18.222.78.21'
```

* * *

### How to Interpret the Output

The output for each command will look something like this (this is an example):

```
wp_posts:guid:
1838:http://18.222.78.21/?page_id=1838

wp_posts:post_content:
1788: ... some content ... <a href="http://18.222.78.21/some-page">Click Here</a> ... more content ...
```

*   `wp_posts:guid`: This tells you the `guid` column in the `wp_posts` table contains the bad URL. The number at the beginning (`1838`) is the Post ID.
    
*   `wp_posts:post_content`: This tells you the bad URL was found within the main content of a page or post. The number at the beginning (`1788`) is its Post ID.
    

By reviewing these lists, you will have a complete picture of every item that will be affected by the Better Search Replace tool. Once you're satisfied, you can proceed with the plan we outlined in the previous step.
