# LUKS简介

# 数据分区加密

## 加密分区
在Debian系的系统中使用LUKS，首先要确保你的系统中已经正确安装了**cryptsetup**这个包
```sh
sudo apt update && apt install cryptsetup -y
```
假设我们要加密**/dev/sda5**这个分区，因为这个分区中存放着一些重要的数据，我们不想让其他人可以随意访问他。

首先执行以下命令，这会为你创建一个加密的数据分区：
```sh
sudo cryptsetup -y -v luksFormat /dev/sda5
```
然后系统会提示你输入一个大写的**YES**来确认你的操作，然后会让你输入密码。完成之后，这个分区就已经被加密了。

## 使用加密后的分区
要想使用加密过的分区，首先需要使用之前设置的密码对其解密才行。
```sh
sudo cryptsetup luksOpen /dev/sda5 crypt_data
```
该命令会提示你输入密码解密该分区，成功之后，系统的**/dev/mapper**目录下出现一个**crypt_data**块设备文件，
该文件的文件名就是上一步**luskOpen**时的最后一个参数。

然后就可以对这个快设备文件进行操作，比如建立文件系统、挂载、读写等，就像普通的快设备文件一样。
```sh
sudo mkfs.ext4 /dev/mapper/crypt_data
sudo mount /dev/mapper/crypt_data /mnt
```
**注意，**这个时候**/dev/sda5**这个文件已经不能进行正常的建立文件系统，挂载等操作，而仅仅用来加解密，
相应的文件系统操作都应该对**/dev/mapper**目录下生成的文件来进行。

当进行完数据操作之后，通过以下命令来卸载分区，使分区重新回到加密状态。
```sh
sudo umount /dev/sda5
sudo cryptsetup luksClose cry_data
```
# 根分区加密(不包括/boot)

# 使用kyefile的普通系统分区加密
如果系统分区**/var**是一个单独的分区，假设是/dev/sda5，并且该分区是使用cryptsetup加密过的，
那么应该如何使用keyfile来在开机时自动进行分区解密呢？

### 创建keyfile
首先创建一个keyfile文件。
```sh
sudo dd if=/dev/urandom of=/root/keyfile bs=512 count=4
sudo chmod 0400 /root/keyfile
```

### 用keyfile加密分区
```sh
sudo cryptsetup luksAddKey /dev/sda5 /root/keyfile
```

### 修改`crypttab`文件
最后，修改`/etc/crypttab`文件，在keyfile那一部分加入`/root/keyfile`，`/etc/crypttab`配置文件的结构如下所示：
```text
<target name>	<source device>		<key file>	<options>
```

# 使用keyfile的根分区加密(不包括/boot)
假设你已通过上一节的方法安装了全盘加密(不包括/boot)的Debian系统，那么接下来介绍如何使用keyfile来进行全盘加密的自动解密。
用来解密的keyfile被存储在initramfs中。下面是具体步骤。

假设下面是现在的系统布局状况：

* /dev/sda1 /boot 不加密
* /dev/sda2 / 加密

### 生成keyfile
首先生成一段随机的二进制数据来作为我们的keyfile
```sh
sudo dd if=/dev/urandom of=/root/keyfile bs=512 count=4
sudo hmod 0400 /root/keyfile
```
### 使用keyfile加密根分区
然后将使用这个keyfile来加密系统分区
```sh
sudo cryptsetup luksAddKey /dev/sda2 /root/keyfile --key-slot 1
```
可以通过以下命令来进一步查看系统分区的加密状况
```sh
sudo cryptsetup luksDump /dev/sda2
```
### initramfs中加载keyfile的脚本
创建一个新的脚本，用来在initramfs中将这个keyfile导入系统。
```sh
sudo touch /lib/cryptsetup/scripts/loadkeyfile.sh
```
以下是`loadkeyfile.sh`文件的内容
```sh
#!/bin/busybox ash
KEY="${1}"
if [ -f "${KEY}" ]; then
  cat "${KEY}"
else
  PASS=/bin/plymouth ask-for-password --prompt="Key not found. Enter LUKS Password: "
echo "${PASS}"
fi
```
给这个文件加上可执行权限：
```sh
sudo chmod +x /lib/cryptsetup/scripts/loadkeyfile.sh
```
### 创建`update-initramfs`hook脚本
然后再创建一个新的shell脚本，这个脚本在系统创建initramfs时被调用，作用是加载keyfile到initramfs中。
```sh
sudo /etc/initramfs-tools/hooks/keyfile-hook.sh
```
`keyfile-hook.sh`的内容：
```sh
#!/bin/sh
PREREQ=""
prereqs() {
  echo "$PREREQ"
}
case "$1" in
  prereqs)
    prereqs
    exit 0
  ;;
esac
. "${CONFDIR}/initramfs.conf"
. /usr/share/initramfs-tools/hook-functions
if [ ! -f "${DESTDIR}/lib/cryptsetup/scripts/loadkeyfile.sh" ]; then
  if [ ! -d "${DESTDIR}/lib/cryptsetup/scripts" ]; then
  mkdir -p "${DESTDIR}/lib/cryptsetup/scripts"
  fi
  cp /lib/cryptsetup/scripts/loadkeyfile.sh ${DESTDIR}/lib/cryptsetup/scripts/
fi
if [ ! -d "${DESTDIR}/root/" ]; then
  mkdir -p ${DESTDIR}/root/
fi
cp /root/keyfile ${DESTDIR}/root/
```
同样地，给这个文件加上可执行权限：
```sh
sudo chmod +x /etc/initramfs-tools/hooks/keyfile-hook.sh
```

### 修改`/etc/crypttab`配置
修改`/etc/crypttab`文件，使之看起来像下面的样子
```text
sda2_crypt UUID=a8604976-269b-4ab1-8ecc-63960f60f008 /root/keyfile luks,discard,noearly,keyscript=/lib/cryptsetup/scripts/loadkeyfile.sh
```
### 重新生成initramfs
最后，需要执行命令来重新生成initramfs，
```sh
sudo update-ininramfs -u -k all
```
这样，重启系统之后，你会发现，此时已经不再需要输入密码了，系统已经使用initramfs中存储的keyfile文件来自动解密了系统分区。

# 将keyfile放到U盘里
可以将keyfile放到U盘里，这样，把U盘插到电脑上，系统在启动时如果检测到U盘里的keyfile，就会自动用该keyfile解密分区并正常启动，
免除了手动输入密码的繁杂，不过当U盘没有插入电脑或者没有读取到keyfile，系统还是会提示用户输入密码来进行解密。

具体的实现方法与上一节类似，不过要事先准备好一个u盘，把keyfile拷贝到这个u盘的第一个分区上，
要注意，这个分区必须是ext分区，否则会挂载失败。

与上一节的区别主要包括一下几个方面，

- loadkeyfile.sh 文件内容不同
- keyfile-hook.sh 文件内容不同
- /etc/crypttab 配置不同

## loadkeyfile.sh 内容

```sh
#!/bin/busybox ash

if [ -f /dev/sdb1 ]; then
  echo "mount the divece containing keyfile ..."
  mount /dev/sdb1 /root/key 2> /dev/null
fi

KEY="${1}"
if [ -f "${KEY}" ]; then
  cat "${KEY}"
else
  echo "FAILED to find suitable USB keychain ..." >&2
  echo -n "Try to enter your password: " >&2
  read -s -r PASSWD < /dev/console
  echo -n "$PASSWD"
fi

if [ -f /dev/sdb1 ]; then
  umount /ev/sdb1 2> /dev/null
fi
```

## keyfile-hook.sh 内容

```sh
#!/bin/sh
PREREQ=""
prereqs() {
  echo "$PREREQ"
}
case "$1" in
  prereqs)
    prereqs
    exit 0
  ;;
esac

. "${CONFDIR}/initramfs.conf"
. /usr/share/initramfs-tools/hook-functions

if [ ! -f "${DESTDIR}/lib/cryptsetup/scripts/loadkeyfile.sh" ]; then
  if [ ! -d "${DESTDIR}/lib/cryptsetup/scripts" ]; then
  mkdir -p "${DESTDIR}/lib/cryptsetup/scripts"
  fi

  cp /lib/cryptsetup/scripts/loadkeyfile.sh ${DESTDIR}/lib/cryptsetup/scripts/
fi
if [ ! -d "${DESTDIR}/root/key" ]; then
    mkdir -p "${DESTDIR}/root/key"
fi
```

## /etc/cryttab 内容

```text
sda2_crypt UUID=563edceb-392c-49f6-9ace-e9b8a6b418d2 /root/key/keyfile luks,keyscript=/lib/cryptsetup/scripts/loadkeyfile.sh
```

# 使用keyfile的全盘加密(包括/boot)