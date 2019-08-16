# ansible-role-aix-etc_configs

An [Ansible](https://github.com/ansible/ansible) role to manage the common configuration files that live in /etc/ on AIX Servers.

## Important Notes

- **This role is opinionated**
  - There are better defaults than the actual IBM defaults, and this role will use said defaults.
  - These differeneces are documented as required in the template descriptions below.
- **Do not run this first-try against a production system**
  - This role changes *a lot* of environment settings.  
  - Run against a experimental environment first.
  - This should be obvious.
- You can view the contents of the default generated files in the `tests/example_templates` directory

## Requirements / Dependencies

- Runs against AIX LPARs.  
  - Also only tested on AIX 7.2
- `JMESPath` must be installed on the Ansible controller
  - `pip install jmespath`
- Ansible Controller must be running Python 3.6+
  - This role uses a custom variable filter `deep_combine`.  
  - This filter is included in the role itself but requires at least Python 3.6 to run (potentially works in older versions, but untested.)

## Role Variables

### Important Variable Notes

- You will need to escape your variables manually
  - ie: `"This is a\ntest"` vs. `\"This is a\\ntest\"` will render in the template as:

```text
This is a
test

"This is a test"
```

### Global level settings

If below is set to False, disable this role completely
`aixetc_enable`: `true`

If below is set to false, don't apply defaults
`aixetc_skip_defaults`: `false`

If below is true, role will reload daemons on config file change
`aixetc_manage_services`: `true`

If below is true, create a backup of the config file when the template is copied
`aixetc_backup`: `true`

If below is defined and is a directory, the configuration files will be created in the given directory instead of the 'standard' location
`aixetc_dir`: `''`

### Template Variables and Defaults

#### /etc/environment

The variable `etc_environment` is a simple list of `key`:`value` pairs.  Each pair will be added to the `/etc/environment` file.  The defaults below are always applied but can be overwritten.

Defaults:

```yaml
etc_environment:
  PATH: /usr/bin:/etc:/usr/sbin:/usr/ucb:/usr/bin/X11:/sbin:/usr/java7_64/jre/bin:/usr/java7_64/bin
  TZ: Etc/UTC
  LANG: C
  LOCPATH: /usr/lib/nls/loc
  NLSPATH: /usr/lib/nls/msg/%L/%N:/usr/lib/nls/msg/%L/%N.cat:/usr/lib/nls/msg/%l.%c/%N:/usr/lib/nls/msg/%l.%c/%N.cat
  LC__FASTMSG: true
  ODMDIR: /etc/objrepos
  CLCMD_PASSTHRU: 1
  EXTENDED_HISTORY: ON
```

#### /etc/motd

Default:

```yaml
etc_motd:
  message: |
    *******************************************************************************
                              Important Notice!
    *******************************************************************************

    This system and all information contained herein is company property and access
    is granted for authorised users only.

    Use of this system constitutes your acceptance that use is subject to company
    security policies and standards, and any violation that breaches these
    provisions may result in disciplinary action.

    All users of this system are expected to have read applicable company policies
    before making use of this system.

    Use of this system may be subject to monitoring.  Log off immediately if you do
    not agree to these terms.

    *******************************************************************************
```

#### /etc/ntp.conf

If any variable is equal to `false` the template will include the key name but will comment out the line.  eg: `server: false` will be entered into the file as `#server`.

Defaults:

```yaml
etc_ntpconf:
  server: false
  broadcastclient: true
  slewalways: true
  driftfile: /etc/ntp.drift
  logfile: /etc/ntp.log
  tracefile: /etc/ntp.trace
```

This differs from the AIX defaults, which would be:

```yaml
tbd
```

#### /etc/profile

`etc_profile` has only a single attribute: `file`.  If `file` is set and is a jinja2 file it will be imported:

```yaml
etc_profile:
  file: `path_to_file.j2`
```

If the above `etc_profile.file` is not set, the following 'good practice' template will be used:

```shell
# System wide profile.  All variables set here may be overridden by
# a user's personal .profile file in their $HOME directory.  However,
# all commands here will be executed at login regardless.

trap "" 1 2 3
readonly LOGNAME

# Automatic logout, include in export line if uncommented
TMOUT=900

# The MAILMSG will be printed by the shell every MAILCHECK seconds
# (default 600) if there is mail in the MAIL system mailbox.
MAIL=/usr/spool/mail/$LOGNAME
MAILMSG="[YOU HAVE NEW MAIL]"

# If termdef command returns terminal type (i.e. a non NULL value),
# set TERM to the returned value, else set TERM to default lft.
TERM_DEFAULT=lft
TERM=`termdef`
TERM=${TERM:-$TERM_DEFAULT}

# If LC_MESSAGES is set to "C@lft" and TERM is not set to "lft",
# unset LC_MESSAGES.
if [ "$LC_MESSAGES" = "C@lft" -a "$TERM" != "lft" ]
then
    unset LC_MESSAGES
fi

export LOGNAME TMOUT MAIL MAILMSG TERM

alias manf="man -m -M/opt/freeware/man:/opt/freeware/share/man"
set -o vi


#
# Additional modifications which should be updated
#

function checkmail {
    if [ -s "$MAIL" ] ; then   # This is at Shell startup. In normal
        echo "$MAILMSG"        # operation, the Shell checks
    fi                         # periodically.
}

case $LOGIN in
    root)
        export PS1='${USER}@'"$(hostname -s)":'${PWD} # '
        export PATH=${PATH}:/usr/local/sbin
        checkmail
    ;;
    *)
        export PS1='${USER}@'"$(hostname -s)":'${PWD} $ '
        checkmail
    ;;
esac

trap 1 2 3
```

#### /etc/resolv.conf

Default settings:

```yaml
etc_resolvconf:
  nameservers: []
  domain:
  search:
  options_timeout: 1
  options_attempts: 1
```

The role will fail if `nameservers`, `domain`, and `search` are not set.

AIX default settings:

```yaml
tbd
```

#### /etc/syslog.conf

`etc_syslogconf` contains a dict of dicts:

- The name of each dict is the unique file to send data to:
  - The file will be created if it does not exist using `root:system/640` permissions
  - If the file does not have `root:system/640` permissions they will be set.
- Each dict has two attributes, `msg_src` and `rotation`.
  - `msg_src` contains either a string to be directly translated and used for the msg_src
  - `rotation` is the rotation pattern to use.
  - See [IBM's documentation](https://www.ibm.com/support/knowledgecenter/en/ssw_aix_72/filesreference/syslog.conf.html?origURL=ssw_aix_72/com.ibm.aix.files/syslog.conf.htm) for more information

Default behaviour is to append the user supplied list of settings to the defaults below.  If you do not wish to use these defaults, set `aixetc_syslogconf_defaults` to `false` (default is `true`).

##### Default settings

```yaml
etc_syslogconf:
  /var/log/aso/aso.log:
    msg_src: aso.notice
    rotation: rotate size 1m files 8 compress
  /var/log/aso/aso_process.log:
    msg_src: aso.info
    rotation: rotate size 1m files 8 compress
  /var/log/aso/aso_debug.log:
    msg_src: aso.debug
    rotation: rotate size 32m files 8 compress
  /var/adm/ras/syslog.caa:
    msg_src: caa.debug
    rotation: rotate size 10m files 10 compress
  /var/log/syslog:
    msg_src: daemon,syslog.info;user,mail.notice;kern.warn
    rotation: rotate time 1m files 12 compress
  /var/log/auth.log:
    msg_src: auth.info
    rotation: rotate time 1m files 12 compress
  /var/log/sudo.log:
    msg_src: local2.notice
    rotation: rotate time 1m files 12 compress
  /var/log/errpt.log:
    msg_src: local3.notice
    rotation: rotate time 1m files 12 compress
```

#### /etc/mail/aliases

Uses straight `key: value` pairs.

Defaults:

```yaml
etc_mail_aliases:
  MAILER-DAEMON: root
  postmaster: root
  nobody: /dev/null
```

We also create a symlink from `/etc/mail/aliases` to `/etc/aliases`.

#### /etc/mail/sendmail.cf

Everything in `/etc/mail/sendmail.cf` is default except that the DS relays are templated.

This means that `etc_mail_sendmail` contains a single option, `relays`, which must be a list of domains to add to `sendmail.cf`:

```yaml
etc_mail_sendmail:
  relays:
    - my.simple.domain
    - 'foo.bar.mydomain # some comment'
    - 'baz.bar.mydomain # another comment'
```

will become:

```text
DSmy.simple.domain
DSfoo.bar.mydomain # some comment
DSbaz.bar.mydomain # another comment
```

#### /etc/security/login.cfg

NB:

- All variables are marked with the Jinja filter `safe` to ensure escaping works as intended.
- AIX omits default values but we purposefully include them.
- Default value for password encryption is `SHA512` instead of the aix `crypt`:
  - You'll  need to reset each password to have it updated to the new encryption method
  - /etc/security/passwd file will tell you which passwords need updating
    - [*Support for passwords greater than 8 characters*](https://www-01.ibm.com/support/docview.wss?uid=isg3T1010741)
    - [*MD5 password scrambler 'no longer safe'*](https://www.zdnet.com/article/md5-password-scrambler-no-longer-safe/)

Role options+defaults:

```text
default + custom ports:
    herald = string | default(\"Unauthorized use of this system is prohibited.\\n\\rlogin: \")
    logindelay = int | default(0)
    logindisable = int | default(0)
    logininterval = int | default(0)
    loginreenable = int | default(0)
    logintimes = string | default(\"\")
    pwdprompt = string | default(omit)
    sak_enabled = bool | default(false)
    synonym = string | default(omit)
    usernameecho = bool | default(omit)

usw:
    shells = string | default(/bin/sh,/bin/bsh,/bin/csh,/bin/ksh,/bin/tsh,/bin/ksh93,/usr/bin/sh,/usr/bin/bsh,/usr/bin/csh,/usr/bin/ksh,/usr/bin/tsh,/usr/bin/ksh93,/usr/bin/rksh,/usr/bin/rksh93,/usr/sbin/uucp/uucico,/usr/sbin/sliplogin,/usr/sbin/snappd)
    maxlogins = int | default(32767)
    logintimeout = int | default(60)
    auth_type = choice('STD_AUTH','PAM_AUTH') | default('STD_AUTH')
    maxroles = int(1<i<8) | default(8)
    pwd_algorithm = string | default('ssha512')
    mkhomeatlogin = bool | default(false)
    authcontroldomain = string | default('files')
    unix_passwd_compat = bool | default(false)
    sulogfulldate = bool | default(true)
    efssharedkeys  = bool | default(false)
    domainlessgroups = bool | default(false)
```

Role Changes from AIX Defaults:

The AIX default login.cfg does not include the `default` setting for `herald`.
The following `usw` settings are changed from the AIX defaults:

- `pwd_algorithm` by default is `blank`, we use `ssha512`
- `sulogfulldate` by default is `false`, we use `true`

#### /etc/security/mkuser.default

Defaults:

```yaml
etc_security_mkuserdefault:
  user:
    pgrp: staff
    groups: staff
    shell: /usr/bin/ksh93
    home: /home/$USER
  admin:
    pgrp: system
    groups: system
    shell: /usr/bin/ksh93
    home: /home/$USER
```

NB: The defaults have changed the default shell for new users from `/usr/bin/ksh` to `/usr/bin/ksh93`

## Example Playbooks

### Very few changes from Role defaults

The output files of this playbook can be found in `tests/example_templates/etc/`

```yaml
---
- hosts: localhost
  vars:
    manage_etc: true
    etc_ntpconf:
      server: ntp.internal.domain
    etc_environment:
      TZ: Australia/Perth
    etc_security_logincfg:
      usw:
        shells: /opt/freeware/bin/bash,/bin/sh,/bin/bsh,/bin/csh,/bin/ksh,/bin/tsh,/bin/ksh93,/usr/bin/sh,/usr/bin/bsh,/usr/bin/csh,/usr/bin/ksh,/usr/bin/tsh,/usr/bin/ksh93,/usr/bin/rksh,/usr/bin/rksh93,/usr/sbin/uucp/uucico,/usr/sbin/sliplogin,/usr/sbin/snappd
    etc_resolvconf:
      nameservers:
        - 10.999.16.3
        - 10.999.16.10
      domain: internal.domain
      search: internal.domain
    etc_mail_aliases:
      root: admins@domain.com
      sysadm: admins@domain.com
    etc_mail_sendmailcf:
      relays:
        - smtp-relay.internal.domain
  roles:
  - role: d-little.aixetc
  tasks:
```

## Authors

- *David Little* - *Initial work* - [d-little](https://github.com/d-little/)

## License

MIT

## Acknowledgments

- [Ansible](https://www.ansible.com)
- [Aaron Hall on SO (for deep_combine inspiration)](https://stackoverflow.com/a/24088493/)
- [RichVel on SO (for clever Jinja2 in a task)](https://stackoverflow.com/a/40365013/)
- [JMESPath.org](http://jmespath.org/)
- [JSONFormatter.org](https://jsonformatter.org/)
- [Jinja2 Live Parser](http://jinja.quantprogramming.com/)
- [json2yaml.com](https://www.json2yaml.com/)
