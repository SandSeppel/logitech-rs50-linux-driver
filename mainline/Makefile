KVERSION := $(shell uname -r)
KDIR := /lib/modules/$(KVERSION)/build

obj-m := hid-logitech-hidpp.o

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

install: all
	$(MAKE) -C $(KDIR) M=$(PWD) modules_install
	depmod -a

load: all
	-sudo rmmod hid-logitech-hidpp 2>/dev/null || true
	sudo insmod ./hid-logitech-hidpp.ko

unload:
	-sudo rmmod hid-logitech-hidpp 2>/dev/null || true
