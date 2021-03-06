.. include:: /substitutions.rsti

Performing a Remote Backup of a pfSense Configuration
=====================================================

Use the Auto Config Backup Package
----------------------------------

The easiest choice. Install the :doc:`/backup/autoconfigbackup` package, and
enter the |premium_content_link| information, and rest easy knowing it's being
taken care of without needing to worry. Sit back, have a cup of coffee, and read
on to see what the other guys have to do.

Pull it
-------

2.3.3 and Later
^^^^^^^^^^^^^^^

Changes to the diag_backup.php "Download" button on 2.3.3 and later
require a slight adjustment compared to previous versions.

- Fetch the login form and save the cookies and CSRF token::

    wget -qO- --keep-session-cookies --save-cookies cookies.txt \
      --no-check-certificate https://192.168.1.1/diag_backup.php \
      | grep "name='__csrf_magic'" | sed 's/.*value="\(.*\)".*/\1/' > csrf.txt

- Submit the login form along with the first CSRF token and save the
  second CSRF token (can't reuse the same file) -- now the script is
  logged in and can take action::

    wget -qO- --keep-session-cookies --load-cookies cookies.txt \
      --save-cookies cookies.txt --no-check-certificate \
      --post-data "login=Login&usernamefld=admin&passwordfld=pfsense&__csrf_magic=$(cat csrf.txt)" \
      https://192.168.1.1/diag_backup.php  | grep "name='__csrf_magic'" \
      | sed 's/.*value="\(.*\)".*/\1/' > csrf2.txt

- Submit the download form along with the second CSRF token to save a
  copy of config.xml::

    wget --keep-session-cookies --load-cookies cookies.txt --no-check-certificate \
      --post-data "download=download&donotbackuprrd=yes&__csrf_magic=$(head -n 1 csrf2.txt)" \
      https://192.168.1.1/diag_backup.php -O config-router-`date +%Y%m%d%H%M%S`.xml

2.2.6 through 2.3.2-p1
^^^^^^^^^^^^^^^^^^^^^^

The authentication system on current versions of pfSense requires
multiple steps with wget, including CSRF handling:

- Fetch the login form and save the cookies and CSRF token::

    wget -qO- --keep-session-cookies --save-cookies cookies.txt \
      --no-check-certificate https://192.168.1.1/diag_backup.php \
      | grep "name='__csrf_magic'" | sed 's/.*value="\(.*\)".*/\1/' > csrf.txt

- Submit the login form along with the first CSRF token and save the
  second CSRF token (can't reuse the same file) -- now the script is
  logged in and can take action::

    wget -qO- --keep-session-cookies --load-cookies cookies.txt \
      --save-cookies cookies.txt --no-check-certificate \
      --post-data "login=Login&usernamefld=admin&passwordfld=pfsense&__csrf_magic=$(cat csrf.txt)" \
      https://192.168.1.1/diag_backup.php  | grep "name='__csrf_magic'" \
      | sed 's/.*value="\(.*\)".*/\1/' > csrf2.txt

- Submit the download form along with the second CSRF token to save a
  copy of config.xml::

    wget --keep-session-cookies --load-cookies cookies.txt --no-check-certificate \
      --post-data "Submit=download&donotbackuprrd=yes&__csrf_magic=$(head -n 1 csrf2.txt)" \
      https://192.168.1.1/diag_backup.php -O config-router-`date +%Y%m%d%H%M%S`.xml

2.0.x through 2.2.5
^^^^^^^^^^^^^^^^^^^

Some older releases did not need CSRF handling for this action and could
use a different script::

  # wget -qO/dev/null --keep-session-cookies --save-cookies cookies.txt \
  --post-data 'login=Login&usernamefld=admin&passwordfld=pfsense' \
  --no-check-certificate https://192.168.1.1/diag_backup.php
  # wget --keep-session-cookies --load-cookies cookies.txt \
  --post-data 'Submit=download&donotbackuprrd=yes' https://192.168.1.1/diag_backup.php \
  --no-check-certificate -O config-router-`date +%Y%m%d%H%M%S`.xml

The first line authenticates, and the second line grabs the
configuration. CURL may also be used.

Backing up RRD
^^^^^^^^^^^^^^

To have RRD contents included in the backup, omit the "donotbackuprrd"
variable and its value from the backup URL.

Push it
-------

The details of this approach are covered elsewhere on the web, and it
isn't recommended, but it is possible to make it work over ssh.

- Generate an ssh key for the root pfSense user *without a passphrase*.
  (That's the potentially dangerous part)
- Add a user to a remote system, and add the pfSense root user's new
  public key to its *~/.ssh/authorized_keys* file
- Create a cron job on the pfSense box that would copy
  */cf/conf/config.xml* to the remote system with *scp*
- If this is implemented, be careful to manage the remote users as such
  that they have limited access, perhaps lock them down to a single
  directory to which they can only write the config and do nothing
  else. Use *chroot* if possible.
