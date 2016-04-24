OPTLEVEL = -O1
WARNINGS := -Wall -Wextra

CC = clang-3.5 -std=c11 $(WARNINGS) $(OPTLEVEL) -g
CXX = clang++-3.5 $(WARNINGS) -std=c++11 $(OPTLEVEL) -g 
LINK = llvm-link-3.5
OPT = opt-3.5
NM = llvm-nm-3.5
PYTHON = python3

NM-OPTS := -extern-only -defined-only -just-symbol-name
OPT-OPTS := $(OPTLEVEL)

makefile_path := $(abspath $(lastword $(MAKEFILE_LIST)))
makefile_dir := $(abspath $(dir $(lastword $(MAKEFILE_LIST))))

COMMON_INCLUDE_DIR := -I.
TOOLS_DIR := $(abspath $(makefile_dir)/../tools)

SRC_ROOT_DIR := $(makefile_dir)

OBJ_BASE_NAME := obj
OBJ_ROOT_DIR := $(addsuffix $(OBJ_BASE_NAME),$(dir $(makefile_dir)))

GEN_BASE_NAME := gen
GEN_ROOT_DIR := $(addsuffix $(GEN_BASE_NAME),$(dir $(makefile_dir)))

METADATA_BASE_NAME := .makefile_metadata
METADATA_ROOT_DIR := $(addsuffix $(METADATA_BASE_NAME),$(dir $(makefile_dir)))

# Functions
rwildcard = $(foreach d,$(wildcard $(1)*),$(call rwildcard,$(d)/,$(2)) $(filter $(subst *,%,$(2)),$(d)))
genfromsrc = $(subst $(SRC_ROOT_DIR),$(GEN_ROOT_DIR),$(1))
objfromgen = $(subst $(GEN_ROOT_DIR),$(OBJ_ROOT_DIR),$(1))
metadatafromgen = $(subst $(GEN_ROOT_DIR),$(METADATA_ROOT_DIR),$(1))

all::

# Search the source directory for files
RAW_C_SOURCE := $(call rwildcard,$(CURDIR)/,*.c)
RAW_C_HEADERS := $(call rwildcard,$(CURDIR)/,*.h)
RAW_CXX_SOURCE := $(call rwildcard,$(CURDIR)/,*.cpp)
RAW_CXX_HEADERS := $(call rwildcard,$(CURDIR)/,*.hpp)
RAW_PY_HEADERS := $(call rwildcard,$(CURDIR)/,*.h.py)
RAW_PY_SOURCE := $(call rwildcard,$(CURDIR)/,*.c.py)
RAW_SUBMODULES := $(call rwildcard,$(CURDIR)/,$(RAW_C_HEADERS:.h=) $(RAW_CXX_HEADERS:.hpp=) $(RAW_PY_HEADERS:.h.py=))

# Derive names of generated files
GEN_C_SOURCE := $(call genfromsrc,$(RAW_C_SOURCE))
GEN_C_HEADERS := $(call genfromsrc,$(RAW_C_HEADERS))
GEN_CXX_SOURCE := $(call genfromsrc,$(RAW_CXX_SOURCE))
GEN_CXX_HEADERS := $(call genfromsrc,$(RAW_CXX_HEADERS))
GEN_PY_HEADERS := $(call genfromsrc,$(RAW_PY_HEADERS:.h.py=.h))
GEN_PY_SOURCE := $(call genfromsrc,$(RAW_PY_SOURCE:.c.py=.c))
GEN_SUBMODULES := $(call genfromsrc,$(RAW_SUBMODULES))

# Derive names of bitcode files
C_BITCODE := $(call objfromgen,$(GEN_C_SOURCE:.c=.bc)) $(call objfromgen,$(GEN_PY_SOURCE:.c=.bc))
CXX_BITCODE := $(call objfromgen,$(GEN_CXX_SOURCE:.cpp=.bc))
SUB_BITCODE := $(call objfromgen,$(GEN_SUBMODULES:=.bc))

# Abbreviations
ALL_GENERATED := $(GEN_C_SOURCE) $(GEN_C_HEADERS) $(GEN_CXX_SOURCE) $(GEN_CXX_HEADERS) $(GEN_SUBMODULES) $(GEN_PY_SOURCE) $(GEN_PY_HEADERS)
ALL_BITCODE := $(C_BITCODE) $(CXX_BITCODE) $(SUB_BITCODE)

# Derive names of dependency files
ALL_DEPENDENCY := $(call metadatafromgen,$(GEN_C_SOURCE:.c=.mk) $(GEN_CXX_SOURCE:.cpp=.mk) $(GEN_PY_SOURCE:.c=.mk))

# Local code generators:
# Python generators
# foo.h.py => foo.h
# foo.h.py => foo.c

# Disable built-in rules
.SUFFIXES:
MAKEFLAGS += -r -j 4

.PHONY:	prepare
prepare:
	@mkdir -p $(sort $(dir $(ALL_GENERATED) $(ALL_BITCODE) $(ALL_DEPENDENCY)))
	@mkdir -p $(sort $(TOOLS_DIR) $(RAW_SUBMODULES) $(GEN_SUBMODULES))

EXE := baz.exe

all::	$(ALL_BITCODE) $(TOOLS_DIR)/$(EXE) | prepare

run:	$(TOOLS_DIR)/$(EXE)
	$(TOOLS_DIR)/$(EXE)

# .exe files in the tools dir can be built from bitcode at the top of the obj dir
$(abspath $(TOOLS_DIR))/%.exe:	$(abspath $(OBJ_ROOT_DIR)/%.bc) | prepare
	echo "extern \"C\" int $*_main(int, char**); int main(int argc, char** argv){return $*_main(argc,argv);}" | $(CXX) -xc++ - -xir $^ -o $@

$(GEN_PY_HEADERS):	$(GEN_ROOT_DIR)%.h:	$(SRC_ROOT_DIR)%.h.py | $(PREPARE)
	$(PYTHON) $^ > $@

$(GEN_PY_SOURCE):	$(GEN_ROOT_DIR)%.c:	$(SRC_ROOT_DIR)%.c.py | $(PREPARE)
	$(PYTHON) $^ > $@

$(ALL_BITCODE):	| prepare
$(ALL_GENERATED):	| prepare

define COPYFILE =
rm -f $@ && ln $< $@
endef

$(GEN_C_SOURCE):	$(GEN_ROOT_DIR)%.c:	$(SRC_ROOT_DIR)%.c
	$(COPYFILE)

$(GEN_C_HEADERS):	$(GEN_ROOT_DIR)%.h:	$(SRC_ROOT_DIR)%.h
	$(COPYFILE)

$(GEN_CXX_SOURCE):	$(GEN_ROOT_DIR)%.cpp:	$(SRC_ROOT_DIR)%.cpp
	$(COPYFILE)

$(GEN_CXX_HEADERS):	$(GEN_ROOT_DIR)%.hpp:	$(SRC_ROOT_DIR)%.hpp
	$(COPYFILE)


# Ensure headers are copied before attempting to compile the C & C++ files
.PHONY:	generate_before_compile
generate_before_compile:	$(GEN_C_HEADERS) $(GEN_CXX_HEADERS) $(GEN_PY_HEADERS) | prepare

MKDEPS = -MT $@ -MMD -MP -MF $(METADATA_ROOT_DIR)$*.mk

# Convert C and C++ into bitcode
$(C_BITCODE):	$(OBJ_ROOT_DIR)%.bc:	$(GEN_ROOT_DIR)%.c  | generate_before_compile
	$(CC) -c -emit-llvm $(COMMON_INCLUDE_DIR) $(MKDEPS) $< -o $@

$(CXX_BITCODE):	$(OBJ_ROOT_DIR)%.bc:	$(GEN_ROOT_DIR)%.cpp | generate_before_compile
	$(CXX) -c -emit-llvm $(COMMON_INCLUDE_DIR) $(MKDEPS) $< -o $@

# Extract the bitcode files that are below this directory in the tree
# Remove the common prefix
# Remove any that still contain a path separator
# Reattach the common prefix
containing = $(foreach v,$(2),$(if $(findstring $(1),$(v)),$(v)))
not-containing = $(foreach v,$(2),$(if $(findstring $(1),$(v)),,$(v)))
get-immediate-bitcode-deps = $(addprefix $(1),$(call not-containing,/,$(subst $(1),,$(call containing,$(1),$(ALL_BITCODE)))))

empty :=
space := $(empty) $(empty)
comma := $(empty),$(empty)


.SECONDEXPANSION:
$(SUB_BITCODE): $(OBJ_ROOT_DIR)/%.bc:	$$(call get-immediate-bitcode-deps,$(OBJ_ROOT_DIR)/$$*/)
	$(LINK) $^ | $(OPT) -internalize -internalize-public-api-list=$(subst $(space),$(comma),$(shell $(NM) $(NM-OPTS) $(OBJ_ROOT_DIR)/$*/$(notdir $*).bc)) $(OPT-OPTS) -o $@

# Include any dependency files that exist
-include $(ALL_DEPENDENCY)

valquick:	$(TOOLS_DIR)/$(EXE)
	valgrind $^

valgrind:	$(TOOLS_DIR)/$(EXE)
	valgrind --leak-check=full --track-origins=yes --leak-resolution=high --num-callers=48 $^

clean:
	rm -rf $(OBJ_ROOT_DIR)
	rm -rf $(GEN_ROOT_DIR)
	rm -rf $(METADATA_ROOT_DIR)
	rm -rf $(TOOLS_DIR)
