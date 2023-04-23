MODULE_NAME := rwProcMem37
RESMAN_CORE_OBJS:=sys.o
RESMAN_GLUE_OBJS:=


ifneq ($(KERNELRELEASE),)    
	$(MODULE_NAME)-objs:=$(RESMAN_GLUE_OBJS) $(RESMAN_CORE_OBJS)
	obj-m := rwProcMem37.o
else
	KDIR := ../android-kernel/out/android-msm-pixel-4.9/private/msm-google
all:
	make CC=clang HOSTCC=clang -C $(KDIR) M=$(PWD) modules
clean:    
	rm -f *.ko *.o *.mod.o *.mod.c *.symvers *.order
endif    
