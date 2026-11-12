# آموزش کلون گرفتن از ماشین مجازی در ESXi بدون vCenter

## ۱. آماده‌سازی ماشین Template

ابتدا Rocky Linux را نصب کنید.

- به‌روزرسانی سیستم‌عامل
- نصب VMware Tools و نرم‌افزارهای موردنیاز
- تنظیم CPU، RAM و کارت‌های شبکه
- بررسی و حذف یا Consolidate کردن Snapshotها

### نصب VMware Tools در Rocky Linux

```bash
dnf install -y open-vm-tools
systemctl enable --now vmtoolsd
systemctl status vmtoolsd
```

### خاموش کردن ماشین Template

```bash
shutdown -h now
```

## ۲. فعال‌سازی SSH در ESXi

`Host → Manage → Services → TSM-SSH → Start`

سپس از طریق SSH متصل شوید.

```bash
ssh root@ESXI-IP
```

## ۳. بررسی نسخه ESXi

```bash
vmware -vl
```

## ۴. مشاهده ماشین‌های مجازی

```bash
vim-cmd vmsvc/getallvms
```

## ۵. مشاهده Datastoreها

```bash
ls -lh /vmfs/volumes/
```

## ۶. بررسی فضای ذخیره‌سازی

```bash
esxcli storage filesystem list
df -h /vmfs/volumes/datastore1/
```

## ۷. بررسی وضعیت ماشین Template

```bash
vim-cmd vmsvc/power.getstate 1
```

خروجی مورد انتظار: `Powered off`

## ۸. بررسی Snapshotها

```bash
vim-cmd vmsvc/snapshot.get 1
```

پیش از کلون، ماشین نباید Snapshot فعال داشته باشد.

## ۹. بررسی فایل‌های Template

```bash
cd /vmfs/volumes/datastore1/Template/
ls -lh
```

## ۱۰. بررسی تنظیمات VMX

```bash
cat /vmfs/volumes/datastore1/Template/Template.vmx
```

## ۱۱. ساخت اسکریپت کلون خودکار

```bash
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

```

## ۱۲. دادن مجوز اجرای اسکریپت

```bash
chmod +x /vmfs/volumes/datastore1/clone-vms.sh
```

## ۱۳. ساخت اولین ماشین برای تست

```bash
sh /vmfs/volumes/datastore1/clone-vms.sh 1 1
```

## ۱۴. رفع خطای وجود پوشه قبلی

در صورت مشاهده `SKIPPED: /vmfs/volumes/datastore1/VM-01 already exists.` ابتدا پوشه را بررسی کنید.

```bash
ls -lah /vmfs/volumes/datastore1/VM-01/
```

فقط در صورت خالی بودن پوشه:

```bash
rmdir /vmfs/volumes/datastore1/VM-01
sh /vmfs/volumes/datastore1/clone-vms.sh 1 1
```

## ۱۵. ساخت ۹ ماشین باقی‌مانده

پس از تست موفق ماشین اول:

```bash
sh /vmfs/volumes/datastore1/clone-vms.sh 2 10
```

برای ساخت تا ۲۰ ماشین (پس از بررسی ظرفیت سرور):

```bash
sh /vmfs/volumes/datastore1/clone-vms.sh 1 20
```

## ۱۶. مشاهده ماشین‌های ساخته‌شده

```bash
vim-cmd vmsvc/getallvms
```

## ۱۷. روشن کردن ماشین مجازی

شناسه ۲ نمونه است؛ شناسه واقعی را از مرحله قبل بررسی کنید.

```bash
vim-cmd vmsvc/power.on 2
```

در صورت نمایش Move or Copy، گزینه `I copied it` را انتخاب کنید.

## ۱۸. تغییر Hostname در ماشین کلون‌شده

از این مرحله دستورات داخل Rocky Linux و از کنسول ESXi اجرا می‌شوند؛ شبکه هنوز قطع باشد.

```bash
hostnamectl set-hostname vm-01
```

## ۱۹. تولید Machine ID جدید

```bash
rm -f /etc/machine-id /var/lib/dbus/machine-id
systemd-machine-id-setup
ln -sf /etc/machine-id /var/lib/dbus/machine-id
```

## ۲۰. تولید SSH Key جدید

```bash
rm -f /etc/ssh/ssh_host_*
ssh-keygen -A
systemctl restart sshd
```

## ۲۱. بررسی کارت‌های شبکه

```bash
ip -br a
nmcli connection show
nmcli device status
```

## ۲۲. تنظیم IP اختصاصی

IP، Gateway و DNS هر ماشین را متناسب با شبکه تنظیم کنید.

```bash
nmtui
```

## ۲۳. راه‌اندازی مجدد ماشین

```bash
reboot
```

## ۲۴. فعال‌سازی کارت‌های شبکه

`Virtual Machines → VM-01 → Edit Settings → Network Adapter → Connected → Connect at power on`

## ۲۵. بررسی نهایی فضای Datastore

در SSH سرور ESXi:

```bash
df -h /vmfs/volumes/datastore1/
du -h /vmfs/volumes/datastore1/VM-*/
```

## ۲۶. غیرفعال‌سازی SSH در ESXi پس از اتمام کار

`Host → Manage → Services → TSM-SSH → Stop`

---

**مشخصات محیط اجرا:**

| مشخصات | مقدار |
|---|---|
| سرور | HP DL360 G9 |
| Hypervisor | ESXi 7.0 U3 |
| Guest OS | Rocky Linux 10.2 |
| Template | Template |
| Datastore | datastore1 |
| CPU | 4 vCPU |
| RAM | 4 GB |
| Disk | 50 GB |
| Provisioning | Thin |
| Disk Controller | PVSCSI |
| Network Adapter | VMXNET3 |
| تعداد کلون | 10 |

**نکته:** اسکریپت برای Template تک‌دیسکی همین سرور نوشته شده است. در سایر سرورها نام Datastore، شناسه Template، تعداد دیسک‌ها و مسیر فایل‌های VMX و VMDK باید بررسی شوند.
