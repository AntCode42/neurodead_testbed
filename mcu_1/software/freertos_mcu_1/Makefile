# Makefile — FreeRTOS_ThreadCreation sur NUCLEO-F446RE (STM32F446RET6)
#
# À placer à la racine de ton dossier projet, au même niveau que Inc/ et Src/
# (le dossier FreeRTOS_ThreadCreation issu de STM32CubeF4).
#
# Une seule chose à adapter avant de compiler : CUBE_ROOT ci-dessous, qui doit
# pointer vers la racine de ton clone du dépôt STM32CubeF4 (celui avec le
# dossier Drivers/, Middlewares/, Projects/ à la racine).

CUBE_ROOT = ../../../../../../..

# ---------------------------------------------------------------------------
# Outils
# ---------------------------------------------------------------------------
PREFIX   = arm-none-eabi-
CC       = $(PREFIX)gcc
AS       = $(PREFIX)gcc -x assembler-with-cpp
OBJCOPY  = $(PREFIX)objcopy
SIZE     = $(PREFIX)size

# ---------------------------------------------------------------------------
# Cible
# ---------------------------------------------------------------------------
TARGET   = firmware
BUILD_DIR = build

# ---------------------------------------------------------------------------
# Sources — code applicatif de l'exemple
# ---------------------------------------------------------------------------
C_SOURCES = \
Src/main.c \
Src/stm32f4xx_it.c \
$(CUBE_ROOT)/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/system_stm32f4xx.c

# Startup assembleur spécifique au STM32F446xx
ASM_SOURCES = \
$(CUBE_ROOT)/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/gcc/startup_stm32f446xx.s

# ---------------------------------------------------------------------------
# Sources — HAL (tous les fichiers du driver, approche simple pour démarrer ;
# à affiner plus tard si tu veux ne garder que les périphériques utilisés)
# ---------------------------------------------------------------------------
C_SOURCES += $(wildcard $(CUBE_ROOT)/Drivers/STM32F4xx_HAL_Driver/Src/*.c)

# ---------------------------------------------------------------------------
# Sources — FreeRTOS (noyau + portage Cortex-M4F + gestion tas + wrapper CMSIS)
# ---------------------------------------------------------------------------
FREERTOS_DIR = $(CUBE_ROOT)/Middlewares/Third_Party/FreeRTOS/Source

C_SOURCES += \
$(FREERTOS_DIR)/tasks.c \
$(FREERTOS_DIR)/queue.c \
$(FREERTOS_DIR)/list.c \
$(FREERTOS_DIR)/timers.c \
$(FREERTOS_DIR)/event_groups.c \
$(FREERTOS_DIR)/portable/GCC/ARM_CM4F/port.c \
$(FREERTOS_DIR)/portable/MemMang/heap_4.c \
$(FREERTOS_DIR)/CMSIS_RTOS/cmsis_os.c

# ---------------------------------------------------------------------------
# Includes
# ---------------------------------------------------------------------------
C_INCLUDES = \
-IInc \
-I$(CUBE_ROOT)/Drivers/STM32F4xx_HAL_Driver/Inc \
-I$(CUBE_ROOT)/Drivers/CMSIS/Device/ST/STM32F4xx/Include \
-I$(CUBE_ROOT)/Drivers/CMSIS/Include \
-I$(FREERTOS_DIR)/include \
-I$(FREERTOS_DIR)/CMSIS_RTOS \
-I$(FREERTOS_DIR)/portable/GCC/ARM_CM4F

# ---------------------------------------------------------------------------
# Défines
# ---------------------------------------------------------------------------
C_DEFS = -DSTM32F446xx -DUSE_HAL_DRIVER

# ---------------------------------------------------------------------------
# Flags de compilation — Cortex-M4, FPU matérielle (le F446RE en a une)
# ---------------------------------------------------------------------------
MCU_FLAGS = -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard

CFLAGS  = $(MCU_FLAGS) $(C_DEFS) $(C_INCLUDES) -O0 -g -Wall -fdata-sections -ffunction-sections
ASFLAGS = $(MCU_FLAGS) -g

# ---------------------------------------------------------------------------
# Linker — le script .ld correspond exactement à la disposition mémoire du
# STM32F446RET6 (512 Ko flash, 128 Ko RAM). Copie-le depuis le dossier
# STM32CubeIDE/ de l'exemple s'il n'est pas déjà ici.
# ---------------------------------------------------------------------------
LDSCRIPT = STM32F446RETX_FLASH.ld
LDFLAGS  = $(MCU_FLAGS) -T$(LDSCRIPT) -specs=nano.specs -specs=nosys.specs \
           -Wl,-Map=$(BUILD_DIR)/$(TARGET).map -Wl,--gc-sections

# ---------------------------------------------------------------------------
# Règles
# ---------------------------------------------------------------------------
OBJECTS = $(addprefix $(BUILD_DIR)/,$(notdir $(C_SOURCES:.c=.o)))
OBJECTS += $(addprefix $(BUILD_DIR)/,$(notdir $(ASM_SOURCES:.s=.o)))
vpath %.c $(sort $(dir $(C_SOURCES)))
vpath %.s $(sort $(dir $(ASM_SOURCES)))

all: $(BUILD_DIR)/$(TARGET).elf $(BUILD_DIR)/$(TARGET).bin

$(BUILD_DIR)/%.o: %.c | $(BUILD_DIR)
	$(CC) -c $(CFLAGS) $< -o $@

$(BUILD_DIR)/%.o: %.s | $(BUILD_DIR)
	$(AS) -c $(ASFLAGS) $< -o $@

$(BUILD_DIR)/$(TARGET).elf: $(OBJECTS)
	$(CC) $(MCU_FLAGS) $(OBJECTS) $(LDFLAGS) -o $@
	$(SIZE) $@

$(BUILD_DIR)/$(TARGET).bin: $(BUILD_DIR)/$(TARGET).elf
	$(OBJCOPY) -O binary $< $@

$(BUILD_DIR):
	mkdir -p $@

flash: $(BUILD_DIR)/$(TARGET).bin
	st-flash write $< 0x8000000

clean:
	rm -rf $(BUILD_DIR)

.PHONY: all flash clean
