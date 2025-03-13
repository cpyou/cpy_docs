https://zhuanlan.zhihu.com/p/612324003



1:使用yum方式安装

```text
yum install openldap openldap-clients openldap-servers
```

2:复制一个默认配置到指定目录下,并授权，这一步一定要做，然后再启动服务，不然生产密码时会报错

```text
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
```

3:授权给ldap用户,此用户yum安装时便会自动创建

```text
chown -R ldap /var/lib/ldap/DB_CONFIG
```

4:启动服务，先启动服务，配置后面再进行修改

```text
systemctl start slapd

systemctl enable slapd
```

5、查看状态，正常启动则ok

```text
systemctl status slapd
```

这里就是重点中的重点了，从openldap2.4.23版本开始，所有配置都保存在`/etc/openldap/slapd.d`目录下的`cn=config`文件夹内，不再使用`slapd.conf`作为配置文件。配置文件的后缀为 `ldif`，且每个配置文件都是通过命令自动生成的，任意打开一个配置文件，在开头都会有一行注释，说明此为自动生成的文件，请勿编辑，使用ldapmodify命令进行修改

6:安装openldap后，会有三个命令用于修改配置文件，分别为ldapadd, ldapmodify, ldapdelete，顾名思义就是添加，修改和删除。而需要修改或增加配置时，则需要先写一个ldif后缀的配置文件，然后通过命令将写的配置更新到slapd.d目录下的配置文件中去

7：初始化配置

生成管理员密码,记录下这个密码，{SSHA}这一串，后面需要用到

```text
slappasswd -s 123456

{SSHA}LSgYPTUW4zjGtIVtuZ8cRUqqFRv1tWpE
```

新增修改密码文件,ldif为后缀，文件名随意，不要在/etc/openldap/slapd.d/目录下创建类似文件 生成的文件为需要通过命令去动态修改ldap现有配置，如下，我在家目录下，创建文件

```bash
cd ~

vim changepwd.ldif
----------------------------------------------------------------------
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}LSgYPTUW4zjGtIVtuZ8cRUqqFRv1tWpE
----------------------------------------------------------------------
# 这里解释一下这个文件的内容：
# 第一行执行配置文件，这里就表示指定为 cn=config/olcDatabase={0}config 文件。
# 你到/etc/openldap/slapd.d/目录下就能找到此文件
# 第二行 changetype 指定类型为修改
# 第三行 add 表示添加 olcRootPW 配置项
# 第四行指定 olcRootPW 配置项的值
# 在执行下面的命令前，你可以先查看原本的olcDatabase={0}config文件，
# 里面是没有olcRootPW这个项的，执行命令后，你再看就会新增了olcRootPW项，
# 而且内容是我们文件中指定的值加密后的字符串
```

执行命令，修改ldap配置，通过-f执行文件

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f changepwd.ldif
```

查看`olcDatabase={0}config`内容,新增了一个`olcRootPW`项。

```text
cat /etc/openldap/slapd.d/cn=config/olcDatabase={0}config.ldif
```

**切记不能直接修改/etc/openldap/slapd.d/目录下的配置。**

我们需要向 LDAP 中导入一些基本的 Schema。这些 Schema 文件位于`/etc/openldap/schema/`目录中，schema控制着条目拥有哪些对象类和属性，可以自行选择需要的进行导入，

```text
依次执行下面的命令，导入基础的一些配置,我这里将所有的都导入一下，
其中core.ldif是默认已经加载了的，不用导入
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/collective.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/corba.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/duaconf.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/java.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/misc.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/pmi.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/ppolicy.ldif
```

### 修改域名，新增con.ldif,

这里我自定义的域名为 [http://kx.com](https://link.zhihu.com/?target=http%3A//kx.com)，管理员用户账号为admin。 如果要修改，则修改文件中相应的dc=kx,dc=com为自己的域名，密码切记改成上面重新生成的哪个。

```text
$ vim con.ldif

dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=admin,dc=yaobili,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=kx,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=kx,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}LSgYPTUW4zjGtIVtuZ8cRUqqFRv1tWpE

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=admin,dc=kx,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=admin,dc=kx,dc=com" write by * read
```

执行命令，修改配置

```text
ldapmodify -Y EXTERNAL -H ldapi:/// -f con.ldif
```

最后这里有5个修改，所以执行会输出5行表示成功。