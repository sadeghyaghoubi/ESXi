# راهنمای کلون گروهی ماشین مجازی در ESXi بدون vCenter

مستند اجرایی بر اساس تجربه عملی ساخت کلون از ماشین Template روی یک هاست مستقل ESXi.

## ۱. هدف و انتخاب روش

هدف: ساخت ۱۰ ماشین مستقل از یک ماشین Rocky Linux آماده، بدون نصب vCenter. برای یک هاست مستقل، `vmkfstools` و رجیستر کردن VMX با `vim-cmd` روش مناسبی است. vCenter زمانی ارزش بیشتری دارد که مدیریت متمرکز چند هاست، Template، امکانات کلاستر یا فرایندهای گسترده استقرار موردنیاز باشند.

| روش | نیاز به vCenter | کاربرد |
|---|---|---|
| کپی دستی Datastore | خیر | تعداد کم ماشین |
| vmkfstools | خیر | کلون مستقل دیسک |
| اسکریپت Shell | خیر | ساخت گروهی ماشین‌ها |
| OVF Export/Import | خیر | انتقال یا استقرار ماشین |
| vCenter + Template | بله | مدیریت متمرکز |

## ۲. مشخصات محیط واقعی

| مورد | مقدار |
|---|---|
| سرور | HPE ProLiant DL360 Gen9 |
| ESXi | 7.0 Update 3، build-20036589 |
| Datastore | datastore1 |
| فایل‌سیستم | VMFS-6 |
| ظرفیت / فضای آزاد اولیه | 803.2 / 749.9 GiB |
| ماشین مرجع | Template، VMID=1 |
| سیستم‌عامل | Rocky Linux 10.2 |
| Virtual hardware | vmx-19 |
| CPU / RAM هر VM | 4 vCPU / 4096 MB |
| دیسک | 50 GiB، یک دیسک SCSI |
| کنترلر | PVSCSI |
| شبکه اول | VM Network، VMXNET3 |
| شبکه دوم | Private، VMXNET3 |
| نام کلون‌ها | VM-01 تا VM-10 |
| وضعیت مرجع | خاموش، بدون Snapshot ثبت‌شده |

مسیر مرجع: `/vmfs/volumes/datastore1/Template/`

۱۰ کلون مجموعاً ۴۰ vCPU، ۴۰ GiB RAM پیکربندی‌شده و ۵۰۰ GiB ظرفیت منطقی دیسک خواهند داشت. Thin provisioning مصرف اولیه را کاهش می‌دهد، نه حداکثر مصرف را. فضای swap ماشین‌ها، لاگ‌ها، Snapshotها و رشد آینده را نیز در نظر بگیرید.

## ۳. آماده‌سازی Golden Image

- سیستم‌عامل و نرم‌افزارهای مشترک را نصب کنید.
- VMware Tools یا open-vm-tools را بررسی کنید.
- Snapshotها را در صورت وجود بررسی و Consolidate کنید.
- ماشین مرجع را خاموش کنید و هنگام کلون روشن نکنید.
- از اطلاعات مهم نسخه پشتیبان داشته باشید.

SSH را از Host Client در مسیر `Host > Manage > Services > TSM-SSH > Start` فعال کنید و به ESXi متصل شوید.

## ۴. دستورات بررسی اولیه

```sh
vmware -vl
vim-cmd vmsvc/getallvms
ls -lh /vmfs/volumes/
esxcli storage filesystem list
vim-cmd vmsvc/power.getstate 1
vim-cmd vmsvc/snapshot.get 1
ls -lh /vmfs/volumes/datastore1/Template/
cat /vmfs/volumes/datastore1/Template/Template.vmx
df -h /vmfs/volumes/datastore1/
```

خروجی‌های کلیدی مشاهده‌شده:

```text
VMware ESXi 7.0.3 build-20036589
VMware ESXi 7.0 Update 3
Powered off
Get Snapshot:
1  Template  [datastore1] Template/Template.vmx  fedora64Guest  vmx-19
VMFS-6  803.2G  53.4G  749.9G  7%
```

فایل‌های مهم مرجع:

```text
Template-flat.vmdk   50.0G
Template.vmdk        503 B
Template.nvram       8.5K
Template.vmx         4.0K
Template.vmsd        0 B
Template.vmxf        47 B
```

فایل `Template.vmdk` توصیف‌گر دیسک است و ورودی vmkfstools باید همین فایل باشد، نه `Template-flat.vmdk`.

بخش‌های مرتبط VMX:

```ini
virtualHW.version = "19"
numvcpus = "4"
memSize = "4096"
scsi0.virtualDev = "pvscsi"
scsi0:0.fileName = "Template.vmdk"
ethernet0.virtualDev = "vmxnet3"
ethernet0.networkName = "VM Network"
ethernet1.virtualDev = "vmxnet3"
ethernet1.networkName = "Private"
guestOS = "fedora-64"
```

## ۵. روش دستی (برای درک فرایند)

اگر پوشه مقصد وجود ندارد:

```sh
mkdir /vmfs/volumes/datastore1/VM-01
vmkfstools -i \
  /vmfs/volumes/datastore1/Template/Template.vmdk \
  /vmfs/volumes/datastore1/VM-01/VM-01.vmdk -d thin
```

سپس باید VMX مناسب با مسیر دیسک جدید ایجاد و رجیستر شود یا یک ماشین جدید با دیسک موجود بسازید. برای انجام خودکار تمام مراحل، از اسکریپت زیر استفاده کنید. **روش دستی و خودکار را برای یک مقصد هم‌زمان اجرا نکنید.**

## ۶. اسکریپت کلون خودکار

این نسخه همان اسکریپت مورد استفاده در گفتگو است. برای محیط فعلی با یک دیسک و دو کارت شبکه نوشته شده است. برای ماشین‌های دارای چند دیسک، Snapshot، رمزنگاری یا passthrough باید بازبینی شود.

```sh
cat > /vmfs/volumes/datastore1/clone-vms.sh <<'SCRIPT'
#!/bin/sh
set -e

DATASTORE="/vmfs/volumes/datastore1"
SOURCE="$DATASTORE/Template"
START="${1:-1}"
END="${2:-10}"

if ! vim-cmd vmsvc/power.getstate 1 | grep -q "Powered off"; then
    echo "ERROR: Template must be powered off."
    exit 1
fi

i="$START"
while [ "$i" -le "$END" ]; do
    NAME=$(printf "VM-%02d" "$i")
    DEST="$DATASTORE/$NAME"

    echo "========================================"
    echo "Creating $NAME"
    echo "========================================"

    if [ -e "$DEST" ]; then
        echo "SKIPPED: $DEST already exists."
        i=$((i + 1))
        continue
    fi

    mkdir "$DEST"
    echo "Cloning virtual disk..."
    vmkfstools -i "$SOURCE/Template.vmdk" "$DEST/$NAME.vmdk" -d thin

    echo "Copying VM configuration..."
    sed \
        -e '/^uuid\./d' \
        -e '/^vc\.uuid/d' \
        -e '/^ethernet[01]\.generatedAddress/d' \
        -e '/^ethernet[01]\.startConnected/d' \
        -e '/^sched\.swap\.derivedName/d' \
        -e '/^migrate\.hostlog/d' \
        -e '/^extendedConfigFile/d' \
        -e '/^vmci0\.id/d' \
        -e '/^scsi0\.sasWWID/d' \
        -e '/^vm\.createDate/d' \
        -e '/^sata0:0\.startConnected/d' \
        -e "s/Template\.vmdk/$NAME.vmdk/g" \
        -e "s/Template\.nvram/$NAME.nvram/g" \
        -e "s/^displayName = .*/displayName = \"$NAME\"/" \
        "$SOURCE/Template.vmx" > "$DEST/$NAME.vmx"

    cp "$SOURCE/Template.nvram" "$DEST/$NAME.nvram"

    cat >> "$DEST/$NAME.vmx" <<CONFIG

uuid.action = "create"
ethernet0.startConnected = "FALSE"
ethernet1.startConnected = "FALSE"
sata0:0.startConnected = "FALSE"
CONFIG

    echo "Registering $NAME..."
    VMID=$(vim-cmd solo/registervm "$DEST/$NAME.vmx")
    echo "SUCCESS: $NAME registered with ID: $VMID"
    i=$((i + 1))
done

echo "========================================"
echo "Clone operation completed."
echo "========================================"
vim-cmd vmsvc/getallvms
SCRIPT

chmod +x /vmfs/volumes/datastore1/clone-vms.sh
```

اسکریپت دیسک مستقل Thin می‌سازد، تنظیمات CPU/RAM و شبکه را حفظ می‌کند، شناسه‌های قبلی را از VMX حذف می‌کند، اتصال اولیه شبکه و ISO را غیرفعال می‌کند و ماشین را بدون روشن کردن رجیستر می‌کند. پوشه‌های موجود بازنویسی نمی‌شوند. اگر خطایی رخ دهد ممکن است پوشه ناقص باقی بماند.

## ۷. اجرای آزمایشی و ساخت گروهی

ابتدا یک ماشین:

```sh
sh /vmfs/volumes/datastore1/clone-vms.sh 1 1
vim-cmd vmsvc/getallvms
```

خروجی نمونه مورد انتظار (شناسه واقعی ممکن است متفاوت باشد):

```text
Creating VM-01
Cloning virtual disk...
Clone: 100% done.
Copying VM configuration...
Registering VM-01...
SUCCESS: VM-01 registered with ID: 2
```

در Host Client ماشین اول را روشن کنید. اگر پیام `Did you move or copy this virtual machine?` نمایش داده شد، `I copied it` را انتخاب کنید. بوت، دیسک و شبکه را بررسی کنید. سپس:

```sh
sh /vmfs/volumes/datastore1/clone-vms.sh 2 10
```

برای ایجاد همه از ابتدا (پوشه‌های موجود رد می‌شوند):

```sh
sh /vmfs/volumes/datastore1/clone-vms.sh 1 10
```

## ۸. خطای واقعی و راه‌حل انجام‌شده

در اجرای اول اسکریپت:

```text
Creating VM-01
SKIPPED: /vmfs/volumes/datastore1/VM-01 already exists.
Clone operation completed.
```

فهرست ماشین‌ها فقط Template را نمایش می‌داد. بررسی پوشه:

```sh
ls -lh /vmfs/volumes/datastore1/VM-01/
```

خروجی واقعی:

```text
total 0
```

چون پوشه خالی بود، دستور امن زیر اجرا شد (rmdir فقط پوشه خالی را حذف می‌کند):

```sh
rmdir /vmfs/volumes/datastore1/VM-01
sh /vmfs/volumes/datastore1/clone-vms.sh 1 1
vim-cmd vmsvc/getallvms
```

اگر پوشه خالی نیست، آن را بدون بررسی حذف نکنید. در صورت وجود VMX و دیسک سالم، امکان رجیستر مستقیم وجود دارد. اگر پوشه مربوط به تلاش ناموفق است و هیچ VM ثبت‌شده‌ای به آن وابسته نیست، می‌توان پس از بررسی آن را با نام یکتا آرشیو کرد:

```sh
mv /vmfs/volumes/datastore1/VM-01 /vmfs/volumes/datastore1/VM-01-backup
```

پیش از mv مطمئن شوید مقصد پشتیبان وجود ندارد. علت دقیق ایجاد پوشه اولیه مشخص نشد؛ خالی بودن آن تأیید شد و کاربر موفقیت عملیات را گزارش کرد.

## ۹. بررسی نهایی روی ESXi

```sh
vim-cmd vmsvc/getallvms
du -h /vmfs/volumes/datastore1/VM-*/
df -h /vmfs/volumes/datastore1/
```

ساختار مورد انتظار:

```text
datastore1/
├── Template/
├── clone-vms.sh
├── VM-01/
├── VM-02/
├── VM-03/
├── VM-04/
├── VM-05/
├── VM-06/
├── VM-07/
├── VM-08/
├── VM-09/
└── VM-10/
```

## ۱۰. شخصی‌سازی Rocky Linux بعد از کلون

این دستورات **داخل ماشین جدید Rocky Linux** اجرا می‌شوند، نه ESXi و نه Template. از کنسول استفاده کنید و شبکه را تا رفع تداخل هویتی و IP قطع نگه دارید.

نمونه برای VM-01:

```bash
hostnamectl set-hostname vm-01
```

برای تولید Machine ID جدید، در صورت وجود فایل قدیمی مستقل D-Bus آن را نیز بررسی کنید. روش زیر فایل قدیمی را کنار می‌گذارد و D-Bus را به شناسه systemd متصل می‌کند؛ سپس ریبوت لازم است:

```bash
rm -f /etc/machine-id /var/lib/dbus/machine-id
systemd-machine-id-setup
mkdir -p /var/lib/dbus
ln -s /etc/machine-id /var/lib/dbus/machine-id
```

اگر openssh-server نصب است، کلیدهای میزبان را مجدداً تولید کنید:

```bash
rm -f /etc/ssh/ssh_host_*
ssh-keygen -A
```

بررسی شبکه:

```bash
ip -br a
nmcli connection show
nmcli device status
```

IP، Gateway و DNS هر شبکه را مطابق طرح آدرس‌دهی واقعی تنظیم کنید. در صورت اتصال پروفایل NetworkManager به MAC قبلی، تنظیمات پروفایل را اصلاح کنید. سپس:

```bash
reboot
```

بعد از تنظیم IPهای یکتا و ریبوت، اتصال کارت‌های موردنیاز را در ESXi فعال کنید: `Connected` و `Connect at power on`.

بررسی نهایی داخل هر ماشین:

```bash
hostnamectl
cat /etc/machine-id
ip -br a
ip route
```

علاوه بر موارد بالا، هر نرم‌افزار دارای شناسه اختصاصی (مانند agentهای مانیتورینگ، Filebeat یا عضویت کلاستر) باید مطابق مستندات خودش شخصی‌سازی شود. برای Windows از Sysprep استفاده کنید؛ دستورات لینوکس برای Windows کاربرد ندارند.

## ۱۱. چک‌لیست استفاده مجدد

- [ ] نام و مسیر Template و VMID بررسی شد.
- [ ] ماشین خاموش است و Snapshot یا زنجیره دیسک فعال ندارد.
- [ ] تعداد و نام دیسک‌ها و کنترلر بررسی شد.
- [ ] فضای Datastore و RAM کافی است.
- [ ] پوشه مقصد وجود ندارد یا وضعیت آن مشخص است.
- [ ] یک کلون آزمایشی ساخته و بوت شد.
- [ ] UUID و MAC هر دو NIC یکتا هستند.
- [ ] Hostname، Machine ID، کلید SSH و IP یکتا هستند.
- [ ] سایر ماشین‌ها ساخته و بررسی شدند.
- [ ] در صورت عدم نیاز، سرویس SSH هاست متوقف شد.

## ۱۲. وضعیت و محدودیت‌های مستند

کاربر در پایان گفتگو اعلام کرد عملیات کامل انجام شده است. خروجی نهایی فهرست ۱۰ ماشین و بررسی مستقل بوت یا شبکه تک‌تک ماشین‌ها ارائه نشده است. این مستند بازسازی اجرایی کل گفتگو است، نه رونوشت کلمه‌به‌کلمه؛ بخش Machine ID نسبت به دستور اولیه کامل‌تر شده است. اسکریپت اصلی بدون تغییر عملکردی حفظ شده و برای استفاده مجدد باید پیش‌نیازهای محیط جدید بررسی شوند.

مرجع معرفی‌شده در گفتگو: [Broadcom Knowledge Base – vmkfstools](https://knowledge.broadcom.com/external/article/343140).
