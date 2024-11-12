# Nuke built-in rules and variables.
MAKEFLAGS += -rR
.SUFFIXES:

# This is the name that our final executable will have.
# Change as needed.
override OUTPUT := efi-template

# Convenience macro to reliably declare user overridable variables.
override USER_VARIABLE = $(if $(filter $(origin $(1)),default undefined),$(eval override $(1) := $(2)))

# Target architecture to build for. Default to x86_64.
$(call USER_VARIABLE,ARCH,x86_64)

# Check if the architecture is supported.
ifeq ($(filter $(ARCH),aarch64 loongarch64 riscv64 x86_64),)
    $(error Architecture $(ARCH) not supported)
endif

# Default user QEMU flags. These are appended to the QEMU command calls.
$(call USER_VARIABLE,QEMUFLAGS,-m 2G)

# User controllable C compiler command.
$(call USER_VARIABLE,CC,cc)

# User controllable archiver command.
$(call USER_VARIABLE,AR,ar)

# User controllable linker command.
$(call USER_VARIABLE,LD,ld)

# User controllable objcopy command.
$(call USER_VARIABLE,OBJCOPY,objcopy)

# User controllable C flags.
$(call USER_VARIABLE,CFLAGS,-g -O2 -pipe)

# User controllable C preprocessor flags. We set none by default.
$(call USER_VARIABLE,CPPFLAGS,)

ifeq ($(ARCH),x86_64)
    # User controllable nasm flags.
    $(call USER_VARIABLE,NASMFLAGS,-F dwarf -g)
endif

# User controllable linker flags. We set none by default.
$(call USER_VARIABLE,LDFLAGS,)

# Ensure the dependencies have been obtained.
ifeq ($(shell ( ! test -d freestnd-c-hdrs-0bsd || ! test -d cc-runtime || ! test -d nyu-efi ); echo $$?),0)
    $(error Please run the ./get-deps script first)
endif

# Check if CC is Clang.
override CC_IS_CLANG := $(shell ! $(CC) --version 2>/dev/null | grep 'clang' >/dev/null 2>&1; echo $$?)

# Save user CFLAGS and CPPFLAGS before we append internal flags.
override USER_CFLAGS := $(CFLAGS)
override USER_CPPFLAGS := $(CPPFLAGS)

# Internal C flags that should not be changed by the user.
override CFLAGS += \
    -Wall \
    -Wextra \
    -std=gnu11 \
    -nostdinc \
    -ffreestanding \
    -fno-stack-protector \
    -fno-stack-check \
    -fshort-wchar \
    -fno-lto \
    -fPIE \
    -ffunction-sections \
    -fdata-sections

# Internal C preprocessor flags that should not be changed by the user.
override CPPFLAGS := \
    -I src \
    -I nyu-efi/inc \
    -isystem freestnd-c-hdrs-0bsd \
    $(CPPFLAGS) \
    -MMD \
    -MP

ifeq ($(ARCH),x86_64)
    # Internal nasm flags that should not be changed by the user.
    override NASMFLAGS += \
        -Wall
endif

# Architecture specific internal flags.
ifeq ($(ARCH),x86_64)
    ifeq ($(CC_IS_CLANG),1)
        override CC += \
            -target x86_64-unknown-none
    endif
    override CFLAGS += \
        -m64 \
        -march=x86-64 \
        -mno-80387 \
        -mno-mmx \
        -mno-sse \
        -mno-sse2 \
        -mno-red-zone
    override LDFLAGS += \
        -m elf_x86_64
    override NASMFLAGS += \
        -f elf64
endif
ifeq ($(ARCH),aarch64)
    ifeq ($(CC_IS_CLANG),1)
        override CC += \
            -target aarch64-unknown-none
    endif
    override CFLAGS += \
        -mgeneral-regs-only
    override LDFLAGS += \
        -m aarch64elf
endif
ifeq ($(ARCH),riscv64)
    ifeq ($(CC_IS_CLANG),1)
        override CC += \
            -target riscv64-unknown-none
        override CFLAGS += \
            -march=rv64imac
    else
        override CFLAGS += \
            -march=rv64imac_zicsr_zifencei
    endif
    override CFLAGS += \
        -mabi=lp64 \
        -mno-relax
    override LDFLAGS += \
        -m elf64lriscv \
        --no-relax
endif
ifeq ($(ARCH),loongarch64)
    ifeq ($(CC_IS_CLANG),1)
        override CC += \
            -target loongarch64-unknown-none
    endif
    override CFLAGS += \
        -march=loongarch64 \
        -mabi=lp64s
    override LDFLAGS += \
        -m elf64loongarch \
        --no-relax
endif

# Internal linker flags that should not be changed by the user.
override LDFLAGS += \
    -nostdlib \
    -pie \
    -z text \
    -z max-page-size=0x1000 \
    -gc-sections \
    -T nyu-efi/src/elf_$(ARCH)_efi.lds

# Use "find" to glob all *.c, *.S, and *.asm files in the tree and obtain the
# object and header dependency file names.
override CFILES := $(shell cd src && find -L * -type f -name '*.c')
override ASFILES := $(shell cd src && find -L * -type f -name '*.S')
ifeq ($(ARCH),x86_64)
override NASMFILES := $(shell cd src && find -L * -type f -name '*.asm')
endif
override OBJ := $(addprefix obj-$(ARCH)/,$(CFILES:.c=.c.o) $(ASFILES:.S=.S.o))
ifeq ($(ARCH),x86_64)
override OBJ += $(addprefix obj-$(ARCH)/,$(NASMFILES:.asm=.asm.o))
endif
override HEADER_DEPS := $(addprefix obj-$(ARCH)/,$(CFILES:.c=.c.d) $(ASFILES:.S=.S.d))

# Default target.
.PHONY: all
all: bin-$(ARCH)/$(OUTPUT).efi

# Rules to build the nyu-efi objects we need.
nyu-efi/src/crt0-efi-$(ARCH).S.o: nyu-efi

nyu-efi/src/reloc_$(ARCH).c.o: nyu-efi

.PHONY: nyu-efi
nyu-efi:
	$(MAKE) -C nyu-efi/src -f nyu-efi.mk \
		ARCH="$(ARCH)" \
		CC="$(CC)" \
		CFLAGS="$(USER_CFLAGS) -nostdinc" \
		CPPFLAGS="$(USER_CPPFLAGS) -isystem ../../freestnd-c-hdrs-0bsd"

# Link rules for building the C compiler runtime.
cc-runtime-$(ARCH)/cc-runtime.a: cc-runtime/*
	rm -rf cc-runtime-$(ARCH)
	cp -r cc-runtime cc-runtime-$(ARCH)
	$(MAKE) -C cc-runtime-$(ARCH) -f cc-runtime.mk \
		CC="$(CC)" \
		AR="$(AR)" \
		CFLAGS="$(CFLAGS)" \
		CPPFLAGS='-isystem ../freestnd-c-hdrs-0bsd -DCC_RUNTIME_NO_FLOAT'

# Rule to convert the final ELF executable to a .EFI PE executable.
bin-$(ARCH)/$(OUTPUT).efi: bin-$(ARCH)/$(OUTPUT) GNUmakefile
	mkdir -p "$$(dirname $@)"
	$(OBJCOPY) -O binary $< $@
	dd if=/dev/zero of=$@ bs=4096 count=0 seek=$$(( ($$(wc -c < $@) + 4095) / 4096 )) 2>/dev/null

# Link rules for the final executable.
bin-$(ARCH)/$(OUTPUT): GNUmakefile nyu-efi/src/elf_$(ARCH)_efi.lds nyu-efi/src/crt0-efi-$(ARCH).S.o nyu-efi/src/reloc_$(ARCH).c.o $(OBJ) cc-runtime-$(ARCH)/cc-runtime.a
	mkdir -p "$$(dirname $@)"
	$(LD) nyu-efi/src/crt0-efi-$(ARCH).S.o nyu-efi/src/reloc_$(ARCH).c.o $(OBJ) cc-runtime-$(ARCH)/cc-runtime.a $(LDFLAGS) -o $@

# Include header dependencies.
-include $(HEADER_DEPS)

# Compilation rules for *.c files.
obj-$(ARCH)/%.c.o: src/%.c GNUmakefile
	mkdir -p "$$(dirname $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

# Compilation rules for *.S files.
obj-$(ARCH)/%.S.o: src/%.S GNUmakefile
	mkdir -p "$$(dirname $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

ifeq ($(ARCH),x86_64)
# Compilation rules for *.asm (nasm) files.
obj-$(ARCH)/%.asm.o: src/%.asm GNUmakefile
	mkdir -p "$$(dirname $@)"
	nasm $(NASMFLAGS) $< -o $@
endif

# Rules to download the UEFI firmware per architecture for testing.
ovmf/ovmf-code-$(ARCH).fd:
	mkdir -p ovmf
	curl -Lo $@ https://github.com/osdev0/edk2-ovmf-nightly/releases/latest/download/ovmf-code-$(ARCH).fd
	case "$(ARCH)" in \
		aarch64) dd if=/dev/zero of=$@ bs=1 count=0 seek=67108864 2>/dev/null;; \
		riscv64) dd if=/dev/zero of=$@ bs=1 count=0 seek=33554432 2>/dev/null;; \
	esac

ovmf/ovmf-vars-$(ARCH).fd:
	mkdir -p ovmf
	curl -Lo $@ https://github.com/osdev0/edk2-ovmf-nightly/releases/latest/download/ovmf-vars-$(ARCH).fd
	case "$(ARCH)" in \
		aarch64) dd if=/dev/zero of=$@ bs=1 count=0 seek=67108864 2>/dev/null;; \
		riscv64) dd if=/dev/zero of=$@ bs=1 count=0 seek=33554432 2>/dev/null;; \
	esac

# Rules for running our executable in QEMU.
.PHONY: run
run: all ovmf/ovmf-code-$(ARCH).fd ovmf/ovmf-vars-$(ARCH).fd
	mkdir -p boot/EFI/BOOT
ifeq ($(ARCH),x86_64)
	cp bin-$(ARCH)/$(OUTPUT).efi boot/EFI/BOOT/BOOTX64.EFI
	qemu-system-$(ARCH) \
		-M q35 \
		-drive if=pflash,unit=0,format=raw,file=ovmf/ovmf-code-$(ARCH).fd,readonly=on \
		-drive if=pflash,unit=1,format=raw,file=ovmf/ovmf-vars-$(ARCH).fd \
		-drive file=fat:rw:boot \
		$(QEMUFLAGS)
endif
ifeq ($(ARCH),aarch64)
	cp bin-$(ARCH)/$(OUTPUT).efi boot/EFI/BOOT/BOOTAA64.EFI
	qemu-system-$(ARCH) \
		-M virt \
		-cpu cortex-a72 \
		-device ramfb \
		-device qemu-xhci \
		-device usb-kbd \
		-device usb-mouse \
		-drive if=pflash,unit=0,format=raw,file=ovmf/ovmf-code-$(ARCH).fd,readonly=on \
		-drive if=pflash,unit=1,format=raw,file=ovmf/ovmf-vars-$(ARCH).fd \
		-drive file=fat:rw:boot \
		$(QEMUFLAGS)
endif
ifeq ($(ARCH),riscv64)
	cp bin-$(ARCH)/$(OUTPUT).efi boot/EFI/BOOT/BOOTRISCV64.EFI
	qemu-system-$(ARCH) \
		-M virt \
		-cpu rv64 \
		-device ramfb \
		-device qemu-xhci \
		-device usb-kbd \
		-device usb-mouse \
		-drive if=pflash,unit=0,format=raw,file=ovmf/ovmf-code-$(ARCH).fd,readonly=on \
		-drive if=pflash,unit=1,format=raw,file=ovmf/ovmf-vars-$(ARCH).fd \
		-drive file=fat:rw:boot \
		$(QEMUFLAGS)
endif
ifeq ($(ARCH),loongarch64)
	cp bin-$(ARCH)/$(OUTPUT).efi boot/EFI/BOOT/BOOTLOONGARCH64.EFI
	qemu-system-$(ARCH) \
		-M virt \
		-cpu la464 \
		-device ramfb \
		-device qemu-xhci \
		-device usb-kbd \
		-device usb-mouse \
		-drive if=pflash,unit=0,format=raw,file=ovmf/ovmf-code-$(ARCH).fd,readonly=on \
		-drive if=pflash,unit=1,format=raw,file=ovmf/ovmf-vars-$(ARCH).fd \
		-drive file=fat:rw:boot \
		$(QEMUFLAGS)
endif
	rm -rf boot

# Remove object files and the final executable.
.PHONY: clean
clean:
	$(MAKE) -C nyu-efi/src -f nyu-efi.mk ARCH="$(ARCH)" clean
	rm -rf bin-$(ARCH) obj-$(ARCH) cc-runtime-$(ARCH)

# Remove everything built and generated including downloaded dependencies.
.PHONY: distclean
distclean:
	rm -rf bin-* obj-* freestnd-c-hdrs-0bsd cc-runtime* nyu-efi ovmf
