TARGET := origin
TARGET_DIR := $(abspath ../../target)
TARGET_TRIPLE := riscv64gc-unknown-none-elf
MODE := release
PLATFORM_NAME := riscv64-qemu-virt
TARGET_ELF := $(TARGET_DIR)/$(TARGET_TRIPLE)/$(MODE)/$(TARGET)
LD_SCRIPT := $(TARGET_DIR)/$(TARGET_TRIPLE)/$(MODE)/linker_$(PLATFORM_NAME).lds

all: $(TARGET) FORCE

$(TARGET): $(TARGET_ELF)
	@rust-objcopy --binary-architecture=riscv64 --strip-all -O binary $< $@

$(TARGET_ELF): $(LD_SCRIPT)
	@AX_PLATFORM=$(PLATFORM_NAME) \
		RUSTFLAGS="-C link-arg=-T$(LD_SCRIPT) -C link-arg=-no-pie -C link-arg=-znostart-stop-gc" \
		cargo build -p $(TARGET) --target $(TARGET_TRIPLE) --release

$(LD_SCRIPT):
	@AX_PLATFORM=$(PLATFORM_NAME) cargo build -p axhal --target $(TARGET_TRIPLE) --release

clean:
	@rm -rf ./$(TARGET)
	@cargo clean -p $(TARGET) --target riscv64gc-unknown-none-elf --release

FORCE:

.PHONY: FORCE
