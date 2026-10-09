KDIR := $(KDIR)
MDIR := $(realpath $(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
ODIR := $(MDIR)
KMI := $(notdir $(patsubst %/,%,$(KDIR)))

ifeq ($(KMI),android17-6.18)
KBUILD_ARGS := M=$(MDIR) MO=$(ODIR)
else
KBUILD_ARGS := M=$(ODIR) src=$(MDIR)
endif

$(info -- KDIR: $(KDIR))
$(info -- MDIR: $(MDIR))
$(info -- OUTPUT DIR: $(ODIR))
$(info -- KMI: $(KMI))
$(info -- KBUILD_ARGS: $(KBUILD_ARGS))

.PHONY: all compdb clean format check-format

# Anti-detection: rename every defined symbol in kernelsu.ko to a random
# per-build prefix so /proc/kallsyms carries no ksu_* fingerprints. Only
# defined symbols are renamed; undefined ones (resolved against vmlinux at
# load time) are left untouched. Requires nm + objcopy. GNU objcopy in
# minimal containers often lacks aarch64 support, so prefer llvm-objcopy
# from the same toolchain as clang.
NM ?= nm
CC_BIN := $(firstword $(CC))
CC_DIR := $(dir $(shell command -v $(CC_BIN) 2>/dev/null))
OBJCOPY ?= $(firstword $(wildcard $(CC_DIR)llvm-objcopy) $(shell command -v llvm-objcopy 2>/dev/null) objcopy)
KSU_SYM_PREFIX := ks$(shell od -An -N4 -tx1 /dev/urandom 2>/dev/null | tr -d ' \n')_

all: check_symbol
	$(MAKE) -C $(KDIR) $(KBUILD_ARGS) modules compile_commands.json -j$(shell nproc)
	./check_symbol $(ODIR)/kernelsu.ko $(KDIR)/vmlinux
	$(NM) --defined-only $(ODIR)/kernelsu.ko | awk 'NF==3 && $$3 !~ /^\$$/ && !seen[$$3]++ {print $$3" $(KSU_SYM_PREFIX)"$$3}' > $(ODIR)/.ksu-symmap
	$(OBJCOPY) --redefine-syms=$(ODIR)/.ksu-symmap -R .BTF $(ODIR)/kernelsu.ko
	rm -f $(ODIR)/.ksu-symmap

compdb:
	python3 $(MDIR)/.vscode/generate_compdb.py -O $(ODIR) $(MDIR)
clean:
	$(MAKE) -C $(KDIR) $(KBUILD_ARGS) clean
	rm check_symbol
check_symbol: tools/check_symbol.c
	$(CC) tools/check_symbol.c -o check_symbol
format:
	find . \( -name "*.c" -o -name "*.h" \) -print0 | xargs -0 clang-format -i
check-format:
	find . \( -name "*.c" -o -name "*.h" \) -print0 | xargs -0 clang-format --dry-run --Werror

# Keep a new line here!! Because someone may append config
