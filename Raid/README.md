جهت بررسی raid یا تغییرات مربوط به raid میتونید از این دستورات استفاده کنید .
برای اینکار یا به esxi باید ssh بزنید و یا تو صفحه کنسول کلید ترکیبی alt + f1 را بزنید تا بتونید وارد shell بشید.
میتونید از دستورات زیر استفاده کنید ، درصورتی که با خطا مواجه شدید احتمالا فایل SSACLI وجود ندارد که فایل مربوطه نیز اضافه شد و میتوانید دانلود کنید

 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl all show config
 
 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 pd 2I:1:8 modify erase
  
 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 ld 6 delete
 
 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 create type=ld drives=1I:1:22,1I:1:23 raid=1
  
 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 create type=ld drives=1I:1:24 raid=0
  
 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 create type=ld drives=1I:1:8,1I:1:9,1I:1:10,1I:1:11 raid=5
  
 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 ld 5 add drives=allunassigned
   
 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 ld 1 modify size=max forced

 /opt/smartstorageadmin/ssacli/bin/ssacli ctrl slot=0 pd 2I:1:8 modify enableeraseddrive 

ssacli ctrl slot=0 ld 1 delete drives=1I:1:2

ssacli ctrl slot=0 ld 1 add drives=1I:1:3

ssacli ctrl slot=0 pd 1I:1:3 modify led=on

ssacli ctrl slot=0 ld 1 modify raid=0

ssacli ctrl slot=0 ld all show detail

=======================

1- lsblk

2- echo 1>/sys/class/block/sdd/device/rescan

3- dnf install -y cloud-utils-growpart

4- growpart -v /dev/sdb 1

5- xfs_growfs /dev/sdb1 
