CC = clang
CXX = clang++
LINK = llvm-link
OPT = opt
NM = llvm-nm

makefile_path := $(abspath $(lastword $(MAKEFILE_LIST)))
makefile_dir := $(abspath $(dir $(lastword $(MAKEFILE_LIST))))

OBJ_BASE_NAME := obj
OBJ_ROOT_DIR := $(addsuffix $(OBJ_BASE_NAME),$(dir $(makefile_dir)))

C_HEADERS := $(wildcard $(CURDIR)/*.h)
C_SOURCE := $(wildcard $(CURDIR)/*.c)

POTENTIAL_SUBMODULES := $(C_HEADERS:.h=)
CONFIRMED_SUBMODULES := $(patsubst %/.,%,$(wildcard $(addsuffix /.,$(POTENTIAL_SUBMODULES))))

RELATIVE_TO_MAKEFILE := $(subst $(makefile_dir),,$(CURDIR))
CURRENT_SRC_DIR := $(CURDIR)/
CURRENT_OBJ_DIR := $(OBJ_ROOT_DIR)$(RELATIVE_TO_MAKEFILE)/

SUBMODULE_BC = $(addprefix $(CURRENT_OBJ_DIR),$(notdir $(CONFIRMED_SUBMODULES:=.bc)))

C_BITCODE := $(addprefix $(CURRENT_OBJ_DIR),$(notdir $(filter-out $(CONFIRMED_SUBMODULES:=.bc),$(C_SOURCE:.c=.bc))))

LIBRARYDIR = $(dir $(CURRENT_OBJ_DIR))
LIBRARY := $(abspath $(LIBRARYDIR)../$(notdir $(patsubst %/,%,$(CURDIR))).bc)


all::

.PHONY:	prepare

prepare:
	@mkdir -p $(OBJ_ROOT_DIR)

all::	$(LIBRARY) | prepare

$(LIBRARY):	$(SUBMODULE_BC) $(C_BITCODE)
	$(LINK) $^ -o $@

$(SUBMODULE_BC):	$(CURRENT_OBJ_DIR)%.bc:
	@mkdir -p $(CURRENT_OBJ_DIR)/$*
	@$(MAKE) --no-print-directory -C $* -f $(makefile_path)

$(C_BITCODE):	$(CURRENT_OBJ_DIR)%.bc: $(CURRENT_SRC_DIR)%.c
	$(CC) -c -emit-llvm $< -o $@

clean:
	rm -rf $(OBJ_ROOT_DIR)
	rm -rf $(LIBRARY)
