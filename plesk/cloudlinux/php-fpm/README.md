# Plesk NGINX/PHP-FPM Integration for CloudLinux CageFS Support Package
This package contains the instructions and RPM packages needed to get the native
NGINX / PHP-FPM support in Plesk 12 to work properly with CageFS.

## What This Package Does
After following the instructions in this file, you will find that CageFS is able
to properly restrict what files PHP scripts run through PHP-FPM are able to see.
You should also see that CloudLinux hosting limits apply to PHP scripts run
through PHP-FPM.

**NOTE:** PHP selector does not work with this method, but since Plesk 12
provides very user-friendly configuration of PHP settings on a per-site basis,
and those settings are compatible with NGINX, this should not be a serious loss.

## Background
By default, Plesk 12 configures sites to be served up through Apache, and using
FastCGI to execute PHP scripts. This configuration works fine with both
CloudLinux and CageFS (except that PHP settings from Plesk are not picked-up
by CageFS users, but that's dealt with in a separate package of this repo).
The only problems with this configuration are that it tends to be slow and uses
more memory than is necessary. It is faster and more secure than Apache's
built-in PHP module, though.

Plesk provides optional NGINX and PHP-FPM packages that support serving up PHP
more quickly, but those packages aren't supported by CloudLinux and CageFS out
of the box. The most obvious symptom is that sites using PHP-FPM aren't subject
to CageFS file limitations or CloudLinux hosting limits.

The reason for this appears to be that in the FastCGI implementation, Apache
enforces hosting limits through `mod_hostinglimits.so`. When Apache encounters
SuexecUserGroup directives (generated by Plesk), the hosting limits plug-in
automatically places the FastCGI processes in the CloudLinux LVE for the user.

The same behavior does not occur for NGINX because NGINX has no knowledge of
CloudLinux or LVE at all, even though the NGINX processes for the site run under
the site owner's user account. It's as if the user was not subject to LVE or
CageFS limits at all.

The fix for this is to modify the `php-fpm` binary (a component provided by one
of the packages of PHP) with patches from CloudLinux to make PHP itself
LVE-aware.

## Disclaimer
This script has been written and tested only with Plesk 12 and CloudLinux Server
release 6.6 (Leonid Kizim). It may work with other versions but it has
not been tested with them.

## Installation of PHP-FPM for Plesk
### Install Plesk Components
1. Log-in to your Plesk server and become `root` (i.e. `sudo -i` or `sudo su`).
2. Run the Plesk installer (`plesk installer` or
   `/usr/local/psa/admin/sbin/autoinstaller`).
3. Press the `ENTER` key on your keyboard though the first four screens.
4. On the fifth screen (`Main components list for Parallels Plesk`), choose
   option 14 (`Web hosting features`):
   
        14. [.] <+> Web hosting features // 8 of 16 components selected

5. On the next screen, select option 14:

        14. [ ] <+> Nginx web server and reverse proxy support // 0 of 2 components selected
        
6. On the following screen, ensure that both nginx options are selected:

        1. [*] Nginx reverse proxy support
        2. [*] php-fpm support for nginx

7. Press the `ENTER` key on your keyboard three times to start installation.

### Enable Plesk Components via the web
8. Once installation is complete, log-in to the Plesk web interface.

9. Under "Server Management" on the left side of the page, select
   "Tools & Settings".

10. Under the "Server Management" section, select "Services Management".

11. Start both the "Reverse Proxy Server (nginx)" and
    "PHP-FPM support for nginx" services.

### Place Sites in CageFS
12. Still in Plesk, under "Server Management" on the left side of the page,
    select "Extensions".
    
13. Select "CageFS". (If not installed, follow CloudLinux docs to [install
    control panel integration](http://docs.cloudlinux.com/installation2.html)).
    
14. Select all of the hosting users desired under the "Disabled" section and
    click the ">>" arrow to enable CageFS for those users.
    
### Configure Sites to Use PHP-FPM
Complete these steps for each site you want to enable PHP-FPM for:

15. Open the desired site subscription in the "Control Panel" (i.e.
    "Power User" mode for the site).

16. Select "Web Server Settings".

17. Ensure that "Process PHP by nginx" is checked.

    **NOTE:** If the site uses any Apache-specific options, such as redirects
    or rewrites in an `.htaccess` file via `mod_rewrite`, you will need to
    convert those rules into appropriate NGINX directives that you provide in
    the `Additional nginx directives` box. (Drupal-specific directives are
    supplied in [a separate package](../../nginx) of this repo).
    
18. Click "OK".

19. Create a file called "test-php.php" in the document root of the site that
    contains the following line:

        <?php phpinfo(); ?>
    
22. Create another file called `test-perms.php` in the document root of the site
    that contains the following lines:
    
        <?php print "In CageFS? " . (file_exists('/var/.cagefs/.cagefs.token') ? 'Yes' : 'No'); ?>
    
20. In a web browser, try to access `test-php.php`. For example, if the site
    domain is "www.example.com", open "http://www.example.com/test-php.php".
    
21. Check the line named "Server API" and ensure that it reads "FPM/FastCGI".
    
22. Still in the browser, try to access `test-perms.php`. For example, if the
    site domain is "www.example.com", open
    "http://www.example.com/test-perms.php".
    
23. Observe that the result is "In CageFS? No". (We'll fix that in the next
    section).
    
## Installation of LVE-aware PHP
### Binary RPM Method
We've already rolled 64-bit RPMs with the CageFS patches for CloudLinux applied to
[release 41 of PHP 5.4.35 from the Atomic Rocket Turtle (ART) repo](http://updates.atomicorp.com/channels/source/php/php-5.4.35-41.art.src.rpm).

You can find our packages under the RPMS folder in the folder with this README.

To install the replacement RPMs, follow these steps:

1. Copy the RedBottle CloudLinux RPMs into a folder on your server.

2. From a command-line logged-in as `root`, change into the folder you created
   in step #1.

3. Run the following command to have Yum replace the PHP 5.4 packages you
   already have installed on your system with the corresponding patched
   packages:

        rpm -qa | grep -e "^php" | grep 5.4 | \
         sed 's/-41.el5.art.x86_64/-541.cloudlinux.redbottle.x86_64.rpm/' | \
         xargs yum install
   
4. Yum should report no errors and ask for your confirmation.
   - If the list of packages to be installed is empty, you don't have PHP 5.4
     installed; you might have an older or newer version of PHP installed
     instead. You can either install PHP 5.4 now using our RPMs, or proceed
     with the Source RPM method below for your version of PHP.
     
   - If Yum reports any dependency errors, follow its instructions to determine
     what to do. (Since dependency errors are often system-dependant, we can't
     really provide appropriate instructions here.)
     
   - If the list of packages looks correct, confirm it to proceed with
     installing the patched packages.
     
5. The install should cause php-fpm to restart automatically to pick up the new
   versions, but just in case, restart it with `service php-fpm restart`.
   
6. In a web browser, try to access the `test-perms.php` script you created in 
   step #22 in the previous section.
    
7. Observe that the result is "In CageFS? Yes".

Congratulations! You should now have NGINX and PHP-FPM working with CageFS. 
    
### Source RPM (aka "Roll Your Own RPM") Method
This option is more hands-on but better if you want more control over what gets
compiled in to PHP, or in case you want to patch a different version of PHP
to work with LVE and CageFS.

For reference, we've included a PHP spec file under the "SPECS" folder that is
in the same folder as this README. You may want to reference it while you
complete this section.

Here are the steps to build your own RPMs from ART sources:

1.  From a command-line logged-in as a user who has sudo access
    (**NOT `root`**), install the development packages needed to compile PHP:
   
        sudo yum install curl-devel gmp-devel httpd-devel pam-devel \
          sqlite-devel libedit-devel libtool-ltdl-devel libc-client-devel \
          cyrus-sasl-devel openldap-devel postgresql-devel unixODBC-devel \
          libxml2-devel firebird-devel net-snmp-devel libxslt-devel \
          libxml2-devel libXpm-devel t1lib-devel tokyocabinet-devel \
          libmcrypt-devel libtidy-devel freetds-devel aspell-devel \
          recode-devel libicu-devel enchant-devel

2.  Find out the URL to the source RPM (SRPM) for your version of PHP from the
    Atomic Repo. Here are links to the two most common versions:
   
    - [PHP 5.3.27](http://updates.atomicorp.com/channels/source/php/php-5.3.27-21.art.src.rpm)   
    - [PHP 5.4.35](http://updates.atomicorp.com/channels/source/php/php-5.4.35-41.art.src.rpm)
     
3.  Pass the URL from step #2 to `rpm -ivh` on the terminal. For example:

        rpm -ivh http://updates.atomicorp.com/channels/source/php/php-5.4.35-41.art.src.rpm
        
4.  You should find an `rpmbuild` folder in the home folder for the user you are
    logged-in as. Change into the `SOURCES` folder of that folder:

        cd ~/rpmbuild/SOURCES
   
5.  Download the PHP patches supplied by CloudLinux:

        wget http://repo.cloudlinux.com/cloudlinux/sources/da/cl-apache-patches.tar.gz
   
6.  Extract the patches into the current folder:

        tar -xvzf cl-apache-patches.tar.gz`

7.  Take note of the patches that were extracted and locate the one appropriate
    for your version of PHP. Some versions will have two versions -- one that
    applies when autoconf is not being used and one that should be used with
    autoconf (it ends with `_autoconf.patch`).
    
    The autoconf version is the correct version to use.

8.  Change into the "SPECS" folder of the `rpmbuild` folder:

        cd ~/rpmbuild/SPECS
   
9.  Open the `php-art.spec` file for editing:

        nano php-art.spec

10. Find the last `Patch:` line in the file before the `BuildRoot:` line.
    For example, for PHP 5.4, it's `Patch100: ssl_read_timeout-5.4.33.patch`.
   
11. Underneath the line you located in step #9, add a patch line that is one
    patch number higher, and reference the appropriate patch from step #7. For
    example, `Patch101: fpm-lve-php5.4_autoconf.patch`.
    
12. Further down in the file, locate the last `%patch` line. For example, for 
    PHP 5.4, it's `%patch91 -p1 -b .remi-oci8`.
   
13. Underneath the line you located in step #12, add a patch line that is one
    patch number higher with the appropriate flags. For example,
    `%patch101 -p1 -b .lve-autoconf`. (The value passed to `-b` is optional and
    arbitrary but is used internally by the patch process).
    
14. Find the "Release:" line that applies to the release of the
    package version you're using (for example, `Release: 41.art`) and then
    modify it as appropriate for your organization's conventions (for example,
    we change "art" to be "redbottle.cloudlinux").
    
    For a smooth installation, you must pick a release number that is higher
    than the one shown. For example, make "41" -> "541". This ensures that Yum
    will willingly upgrade the existing system packages with your new packages.
    
    An example of a completed Release line is:
    
        Release: 541.myorg.cloudlinux
        
15. (Optional) Find the `%changelog` line and add an appropriate note along
    with your name and e-mail address to indicate your version includes LVE
    patches. This will aid with maintenance down the road.
    
16. Change into the rpmbuild folder:

        cd ~/rpmbuild
        
17. Build the RPMS:

        rpmbuild -ba SPECS/php-art.spec
        
18. Once the build completes, with luck you should find your complete RPMS in
    the ~/rpmbuild/RPMS folder. Change into the RPMs folder:
    
        cd ~/rpmbuild/RPMS
        
19. Pick-up at step #2 in the "Binary RPM Method" section above to install
    your new packages. For step #3, be sure to correct the second part of the
    pattern passed to the `sed` command to match the release pattern you
    specified in step #14.
