KVERSION := $(shell uname -r)
KDIR := /lib/modules/$(KVERSION)/build

obj-m := hid-logitech-hidpp.o

# Use: make CC=clang LLVM=1   for clang builds
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
