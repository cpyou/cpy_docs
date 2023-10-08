docker 安装LDAP

```shell
docker run -p 389:389 -p 636:636 --name ldap --detach osixia/openldap

# vim ssp.conf.php
<?php // My SSP configuration
$keyphrase = "mysecret";
$debug = true;
?>

<?php // My SSP configuration
$keyphrase = "mysecret";
$debug = false;
$use_captcha = true;
$ldap_url = "ldap://192.168.3.189:389";
$ldap_binddn = "cn=admin,dc=example,dc=org";
$ldap_bindpw = "admin";
$ldap_base = "ou=People,dc=example,dc=org";
?>


docker run -p 80:80 --name ssp -v $PWD/ssp.conf.php:/var/www/conf/config.inc.local.php -it docker.io/ltbproject/self-service-password:latest

```



```shell
vim db.ldif
## 添加如下信息
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=example,dc=org

dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=example,dc=org

dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}KTdcymNjZVhP/kyRIMNyv/SkJ5wqk4BQ

## 执行添加
ldapmodify -Y EXTERNAL  -H ldapi:/// -f db.ldif

vim base.ldif
## 添加如下信息
dn: dc=example,dc=org
dc: deepcoin
objectClass: top
objectClass: domain

dn: cn=admin,dc=example,dc=org
objectClass: organizationalRole
cn: admin
description: LDAP Manager

dn: ou=People,dc=example,dc=org
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=example,dc=org
objectClass: organizationalUnit
ou: Group

## 添加数据
ldapadd -x -W -D "cn=admin,dc=example,dc=org" -f base.ldif

```



```ini
<?php
$debug = true;
$use_captcha = false;
$ldap_url = "ldap://192.168.3.189:389";
$ldap_starttls = false;
$ldap_binddn = "cn=admin,dc=example,dc=org";
$ldap_bindpw = '****';
$ldap_base = "ou=People,dc=example,dc=org";
$ldap_login_attribute = "uid";
$ldap_fullname_attribute = "cn";
$ldap_filter = "(&(objectClass=person)($ldap_login_attribute={login}))";
$ldap_use_exop_passwd = false;
$ldap_use_ppolicy_control = false;

# Prefix to use for salt with CRYPT
$hash_options['crypt_salt_prefix'] = "$6$";
$hash_options['crypt_salt_length'] = "6";

$shadow_options['shadow_expire_days'] = -1;

# Hash mechanism for password:
# SSHA, SSHA256, SSHA384, SSHA512
# SHA, SHA256, SHA384, SHA512
# SMD5
# MD5
# CRYPT
# ARGON2
# clear (the default)
# auto (will check the hash of current password)
# This option is not used with ad_mode = true
$hash = "SSHA";
$hash_options=[];

$keyphrase = "keyphrase";
# Local password policy
# This is applied before directory password policy
# Minimal length
$pwd_min_length = 8;
# Maximal length
$pwd_max_length = 20;
# Minimal lower characters
$pwd_min_lower = 1;
# Minimal upper characters
$pwd_min_upper = 1;
# Minimal digit characters
$pwd_min_digit = 0;
# Minimal special characters
$pwd_min_special = 0;
# Definition of special characters
$pwd_special_chars = "^a-zA-Z0-9";
# Forbidden characters
#$pwd_forbidden_chars = "@%";
# Don't reuse the same password as currently
$pwd_no_reuse = true;
# Check that password is different than login
$pwd_diff_login = true;
# Check new passwords differs from old one - minimum characters count
$pwd_diff_last_min_chars = 0;
# Forbidden words which must not appear in the password
$pwd_forbidden_words = array();
# Forbidden ldap fields
# Respective values of the user's entry must not appear in the password
# example: $pwd_forbidden_ldap_fields = array('cn', 'givenName', 'sn', 'mail');
$pwd_forbidden_ldap_fields = array();
# Complexity: number of different class of character required
$pwd_complexity = 0;
# use pwnedpasswords api v2 to securely check if the password has been on a leak
$use_pwnedpasswords = false;
# Show policy constraints message:
# always
# never
# onerror
$pwd_show_policy = "never";
# Position of password policy constraints message:
# above - the form
# below - the form
$pwd_show_policy_pos = "above";

# disallow use of the only special character as defined in `$pwd_special_chars` at the beginning and end
$pwd_no_special_at_ends = false;

# Who changes the password?
# Also applicable for question/answer save
# user: the user itself
# manager: the above binddn
#$who_change_password = "user";
$who_change_password = "admin";

# Show extended error message returned by LDAP directory when password is refused
$show_extended_error = false;

## Standard change
# Use standard change form?
$use_change = true;

## SSH Key Change
# Allow changing of sshPublicKey?
$change_sshkey = false;

# What attribute should be changed by the changesshkey action?
$change_sshkey_attribute = "sshPublicKey";

# What objectClass is required for that attribute?
$change_sshkey_objectClass = "ldapPublicKey";

# Ensure the SSH Key submitted uses a type we trust
$ssh_valid_key_types = array('ssh-rsa', 'ssh-dss', 'ecdsa-sha2-nistp256', 'ecdsa-sha2-nistp384', 'ecdsa-sha2-nistp521', 'ssh-ed25519');

# Who changes the sshPublicKey attribute?
# Also applicable for question/answer save
# user: the user itself
# manager: the above binddn
$who_change_sshkey = "user";

# Notify users anytime their sshPublicKey is changed
## Requires mail configuration below
$notify_on_sshkey_change = false;

## Questions/answers
# Use questions/answers?
$use_questions = true;
# Allow to register more than one answer?
$multiple_answers = false;
# Store many answers in a single string attribute
# (only used if $multiple_answers = true)
$multiple_answers_one_str = false;

# Answer attribute should be hidden to users!
$answer_objectClass = "extensibleObject";
$answer_attribute = "info";

# Crypt answers inside the directory
$crypt_answers = true;

# Extra questions (built-in questions are in lang/$lang.inc.php)
# Should the built-in questions be included?
$questions_use_default = true;
#$messages['questions']['ice'] = "What is your favorite ice cream flavor?";

# How many questions must be answered.
#  If = 1: legacy behavior
#  If > 1:
#    this many questions will be included in the page forms
#    this many questions must be set at a time
#    user must answer this many correctly to reset a password
#    $multiple_answers must be true
#    at least this many possible questions must be available (there are only 2 questions built-in)
$questions_count = 1;

# Should the user be able to select registered question(s) by entering only the login?
$question_populate_enable = false;

## Token
# Use tokens?
# true (default)
# false
$use_tokens = true;
# Crypt tokens?
# true (default)
# false
$crypt_tokens = true;
# Token lifetime in seconds
$token_lifetime = "3600";

## Mail
# LDAP mail attribute
$mail_attributes = array("mail");
# Get mail address directly from LDAP (only first mail entry)
# and hide mail input field
# default = false
$mail_address_use_ldap = false;
# Who the email should come from
$mail_from = "***";
$mail_from_name = "Self Service Password";
$mail_signature = "";
# Notify users anytime their password is changed
$notify_on_change = true;
# PHPMailer configuration (see https://github.com/PHPMailer/PHPMailer)
$mail_sendmailpath = '/usr/sbin/sendmail';
$mail_protocol = 'smtp';
$mail_smtp_debug = true;
$mail_debug_format = 'error_log';
$mail_smtp_host = '***';
$mail_smtp_auth = true;
$mail_smtp_user = '***';
$mail_smtp_pass = '***';
$mail_smtp_port = 465;
$mail_smtp_timeout = 5;
$mail_smtp_keepalive = false;
$mail_smtp_secure = 'ssl';  //默认是tls，需要修改成ssl,否则发送邮件不成功
$mail_smtp_autotls = true;
$mail_smtp_options = array();
$mail_contenttype = 'text/plain';
$mail_wordwrap = 0;
$mail_charset = 'utf-8';
$mail_priority = 3;

## SMS
# Use sms
$use_sms = false;

?>
```

问题解决

1. 容器版本的普通用户默认有修改自身密码权限，2.4.44 centos 的默认没有会报“密码被 LDAP 目录服务拒绝”



```shell
slapd -VV
vim updatepass.ldif
```



```ini
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: to attrs=userPassword
       by self =xw
       by anonymous auth
       by * none

olcAccess: to *
       by self write
       by users read
       by * none
```

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f updatepass.ldif 
```



