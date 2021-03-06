Title: Как запустить VM в qemu c UEFI.
Date: 2015-03-31 01:28
Authors: sbog
Slug: libvirt_uefi
Tags: libvirt, qemu, uefi, vm
Lang: ru

Сегодня мне понадобилось потестировать работу линуксового инсталлятора под UEFI.
А материнской платы с EFI под рукой не оказалось (на самом деле нет, но мне было
лень перезагружаться и я просто захотел потестить все в виртуалке). Возник
вопрос - как запустить что-либо в QEMU под UEFI, а не под BIOS? Ответ, как обычно,
нашелся. И звучит он так: OVMF.

Open Virtual Machine Firmware или сокращенно OVMF, это проект, который предоставляет
поддержку UEFI firmware для виртуальных машин. Предоставляет - это значит, что
его надо будет [скачать][0] и [скомпилить][1]. После этого на выходе у вас будет
файл (или два), который надо подсунуть QEMU как firmware.

У меня был Arch Linux и собирать мне было лень, потому как надо было качать то,
се, да и вообще, это же Арч - все уже собрано до нас, подумал я - и не ошибся.
Пакет называется (удивительно, ага) ovmf и доступен с репозиториях. После
установки в `/usr/share/ovmf/` будут лежать файлы с firmware. Я быстренько
накропал [скриптик][2], который spawn-ит виртуальную машину для тестов. И оно
действительно работает.

[0]: https://github.com/tianocore/edk2.git
[1]: http://www.linux-kvm.org/downloads/lersek/ovmf-whitepaper-c770f8c.txt
[2]: https://github.com/sorrowless/common-files/blob/master/home/sbog/common/qvm_new_vm.py
