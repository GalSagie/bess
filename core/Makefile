# Copyright (c) 2014-2017, The Regents of the University of California.
# Copyright (c) 2016-2017, Nefeli Networks, Inc.
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# * Redistributions of source code must retain the above copyright notice, this
# list of conditions and the following disclaimer.
#
# * Redistributions in binary form must reproduce the above copyright notice,
# this list of conditions and the following disclaimer in the documentation
# and/or other materials provided with the distribution.
#
# * Neither the names of the copyright holders nor the names of their
# contributors may be used to endorse or promote products derived from this
# software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.

# Disable all implicit Makefile rules
MAKEFLAGS += --no-builtin-rules
.SUFFIXES: ;

CXX ?= g++
PROTOC ?= protoc

VERBOSE ?= 0

DEPDIR ?= .deps
DEPFLAGS = -MT $@ -MMD -MP -MF $(DEPDIR)/$(@:.o=.d)
$(shell mkdir -p $(DEPDIR) >/dev/null)
$(shell mkdir -p $(DEPDIR)/utils >/dev/null)
$(shell mkdir -p $(DEPDIR)/modules >/dev/null)
$(shell mkdir -p $(DEPDIR)/drivers >/dev/null)
$(shell mkdir -p $(DEPDIR)/hooks >/dev/null)
$(shell mkdir -p $(DEPDIR)/pb >/dev/null)

# 'clang' or 'g++'
CXXCOMPILER := $(shell expr $(word 1, $(shell $(CXX) --version)) : '\(clang\|g++\)')

CXXVERSION := $(shell $(CXX) -dumpversion)

ifeq "$(CXXCOMPILER)" "g++"
ifneq "$(shell printf '$(CXXVERSION)\n5' | sort -V | head -n1)" "5"
$(error g++ 5 or higher is required. Use container_build.py if newer g++ is not available.)
endif
endif

RTE_SDK ?= $(abspath ../deps/dpdk-17.05)
RTE_TARGET ?= $(shell uname -m)-native-linuxapp-gcc
DPDK_LIB ?= dpdk

ifneq ($(wildcard $(RTE_SDK)/$(RTE_TARGET)/*),)
	DPDK_INC_DIR := $(RTE_SDK)/$(RTE_TARGET)/include
	DPDK_LIB_DIR := $(RTE_SDK)/$(RTE_TARGET)/lib
else ifneq ($(wildcard $(RTE_SDK)/build/*),)
	# if the user didn't do "make install" for DPDK
	DPDK_INC_DIR := $(RTE_SDK)/build/include
	DPDK_LIB_DIR := $(RTE_SDK)/build/lib
else ifeq ($(words $(MAKECMDGOALS)),1)
	ifneq ($(MAKECMDGOALS),clean)
	$(error DPDK is not available. \
		Make sure $(abspath $(RTE_SDK)) is available and built)
	endif
endif

CXXARCHFLAGS ?= -march=native
CXXFLAGS += -std=c++11 -g3 -ggdb3 $(CXXARCHFLAGS) \
            -Werror -isystem $(DPDK_INC_DIR) -isystem . -D_GNU_SOURCE \
            -Wall -Wextra -Wcast-align

PERMISSIVE := -Wno-unused-parameter -Wno-missing-field-initializers \
	      -Wno-unused-private-field

# -Wshadow should not be used for g++ 4.x, as it has too many false positives
ifeq "$(shell expr $(CXXCOMPILER) = g++ \& $(CXXVERSION) \< 50000)" "0"
	CXXFLAGS += -Wshadow
endif

# Disable GNU_UNIQUE symbol for g++
ifeq "$(shell expr $(CXXCOMPILER) = g++)" "1"
	CXXFLAGS += -fno-gnu-unique
endif

LDFLAGS += -rdynamic -L$(DPDK_LIB_DIR) -Wl,-rpath=$(DPDK_LIB_DIR) -pthread
ifdef BESS_LINK_DYNAMIC
    LIBS_ALL_SHARED = -Wl,-call_shared
    LIBS_DL_SHARED =
else # Used static libraries
    LIBS_ALL_SHARED =
    LIBS_DL_SHARED = -Wl,-call_shared
    LIBS_LZMA = -llzma
    LDFLAGS += -static-libstdc++
endif

LIBS += -Wl,-non_shared \
	-Wl,--whole-archive -l$(DPDK_LIB) -Wl,--no-whole-archive \
	$(LIBS_ALL_SHARED) \
	-lglog -lgflags -lprotobuf -lgrpc++ -lgrpc \
	-lssl -lcrypto -lunwind $(LIBS_LZMA) -lpcap -lz \
	$(LIBS_DL_SHARED) \
	-ldl

ifdef SANITIZE
	CXXFLAGS += -fsanitize=address -fsanitize=undefined -fno-omit-frame-pointer
	LDFLAGS += -fsanitize=address -fsanitize=undefined
endif

ifdef DEBUG
	CXXFLAGS += --coverage -O0
	LDFLAGS += --coverage
else
	CXXFLAGS += -Ofast -DNDEBUG
endif

-include extra*.mk

PROTO_DIR := $(abspath ../protobuf)
PROTOS := $(wildcard $(PROTO_DIR)/*.proto)
PROTO_SRCS := $(patsubst %.proto,pb/%.pb.cc, $(notdir $(PROTOS)))
PROTO_SRCS += $(patsubst %.proto,pb/%.grpc.pb.cc, $(notdir $(PROTOS)))
PROTO_HEADERS := $(patsubst %.cc,%.h, $(PROTO_SRCS))
PROTOCFLAGS += --proto_path=$(PROTO_DIR) --cpp_out=./pb --grpc_out=./pb --plugin=protoc-gen-grpc=`which grpc_cpp_plugin`

ALL_SRCS := $(wildcard *.cc utils/*.cc modules/*.cc drivers/*.cc hooks/*.cc)

TEST_SRCS := $(filter %_test.cc gtest_main.cc, $(ALL_SRCS))
TEST_OBJS := $(TEST_SRCS:.cc=.o)
TEST_EXEC := $(filter-out gtest_main, $(TEST_OBJS:%.o=%))
TEST_ALL_EXEC := all_test

BENCH_SRCS := $(filter %_bench.cc, $(ALL_SRCS))
BENCH_OBJS := $(BENCH_SRCS:.cc=.o)
BENCH_EXEC := $(BENCH_OBJS:%.o=%)

MODULE_SRCS := $(filter-out $(TEST_SRCS) $(BENCH_SRCS), $(filter modules/%.cc, $(ALL_SRCS)))
MODULE_OBJS := $(MODULE_SRCS:.cc=.o)
MODULE_LIBS := $(MODULE_OBJS:.o=.so)

SRCS := $(filter-out $(TEST_SRCS) $(BENCH_SRCS) $(MODULE_SRCS), $(ALL_SRCS)) $(PROTO_SRCS)
HEADERS := $(wildcard *.h utils/*.h drivers/*.h hooks/*.h)
OBJS := $(SRCS:.cc=.o)

EXEC := bessd

GTEST_DIR := /usr/src/gtest

.PHONY: all clean tags cscope tests benchmarks protobuf

all: $(EXEC) modules tests benchmarks

clean:
	rm -rf $(EXEC) .deps/*.d .deps/*/*.d *_test */*_test *_bench */*_bench \
		*.a pb/*.pb.* *.o */*.o *.so */*.so *.gcov *.gcda *.gcno */*.gcda */*.gcno \
		coverage.info coverage_html

tags:
	@ctags -R *

cscope:
	@rm -f cscope.*
	@find . -name "*.c" -o -name "*.h" > cscope.files
	cscope -b -q -k
	@rm -f cscope.files

tests: $(TEST_OBJS) $(TEST_EXEC) $(TEST_ALL_EXEC)

benchmarks: $(BENCH_OBJS) $(BENCH_EXEC)

protobuf: $(PROTO_SRCS)

modules: protobuf $(MODULE_OBJS) $(MODULE_LIBS)

# Generate version string from the current status of the git working copy
VERSION ?= $(shell git describe --dirty --always --tags 2>/dev/null)

# Default if previous command fails (e.g., when building from a tarball)
DEFAULT_VERSION = unknown

VERSION_LINE = \#define VERSION \"$(or $(VERSION), $(DEFAULT_VERSION))\"
version.h: $(SRCS) $(filter-out version.h, $(HEADERS))
	@echo "$(VERSION_LINE)" > $@

# Make sure version.h is created before its use.
# This is necessary since ./deps/main.d might not be there yet.
main.o: version.h

# This build wrapper takes 4 parameters:
# $(1): build type (CXX, LD, ...)
# $(2): Make target
# $(3): Make prerequisites
# $(4): command
define BUILD
$(2): $(3)
	$$(eval _TYPE := $$(strip $(1)))
	$$(eval _CMD := $$(strip $(4)))
	@if [ $$(VERBOSE) -eq 0 ]; then \
		printf "%-11s %s\n" "[$$(_TYPE)]" "$$@"; \
	else \
		printf "%-11s %s\n" "[$$(_TYPE)]" "$$(_CMD)"; \
	fi
	@if ! $$(_CMD); then \
		echo "Error: \033[0;31m$$@"; \
		echo "\033[0;33m$$(_CMD)\033[0m"; \
		false; \
	fi
endef

$(eval $(call BUILD, \
	PROTOC, \
	pb/%.pb.cc pb/%.pb.h pb/%.grpc.pb.cc pb/%.grpc.pb.h, \
	$(PROTO_DIR)/%.proto, \
	$(PROTOC) $$< $(PROTOCFLAGS)))

$(eval $(call BUILD, \
	CXX, \
	pb/%.pb.o, \
	pb/%.pb.cc, \
	$(CXX) -o $$@ -c $$< $(CXXFLAGS) $(PERMISSIVE) $$(DEPFLAGS)))

$(eval $(call BUILD, \
	MODULE_CXX, \
	modules/%.o, \
	modules/%.cc, \
	$(CXX) -o $$@ -c $$< $(CXXFLAGS) $$(DEPFLAGS) -fPIC))

$(eval $(call BUILD, \
	MODULE_LD, \
	%.so, \
	%.o, \
	$(CXX) -shared -o $$@ $$^ $(LDFLAGS)))

$(eval $(call BUILD, \
	MODULE_TEST_LD, \
	modules/%_test, \
	modules/%_test.o modules/%.o gtest-all.o gtest_main.o bess.a, \
	$(CXX) -o $$@ $$^ $(LDFLAGS) $(LIBS)))

$(eval $(call BUILD, \
	MODULE_BENCH_LD, \
	modules/%_bench, \
	modules/%_bench.o modules/%.o bess.a, \
	$(CXX) -o $$@ $$^ $(LDFLAGS) $(LIBS) -lbenchmark))

$(eval $(call BUILD, \
	CXX, \
	%.o, \
	%.cc $(PROTO_HEADERS), \
	$(CXX) -o $$@ -c $$< $(CXXFLAGS) $$(DEPFLAGS)))

$(eval $(call BUILD, \
	LD, \
	$(EXEC), \
	$(OBJS), \
	$(CXX) -o $$@ $$^ $(LDFLAGS) $(LIBS)))

$(eval $(call BUILD, \
	TEST_CXX, \
	%_test.o, \
	%_test.cc, \
	$(CXX) -o $$@ -c $$< $(CXXFLAGS) $$(DEPFLAGS)))

$(eval $(call BUILD, \
	TEST_LD, \
	%_test, \
	%_test.o gtest-all.o gtest_main.o bess.a, \
	$(CXX) -o $$@ $$^ $(LDFLAGS) $(LIBS)))

$(eval $(call BUILD, \
	TEST_LD, \
	$(TEST_ALL_EXEC), \
	$(TEST_OBJS) $(MODULE_OBJS) gtest-all.o bess.a, \
	$(CXX) -o $$@ $$^ $(LDFLAGS) $(LIBS)))

$(eval $(call BUILD, \
	TEST_CXX, \
	gtest-all.o, \
	$(GTEST_DIR)/src/gtest-all.cc, \
	$(CXX) -o $$@ -c $$< -isystem $(GTEST_DIR) $(CXXFLAGS) $$(DEPFLAGS)))

$(eval $(call BUILD, \
	BENCH_CXX, \
	%_bench.o, \
	%_bench.cc, \
	$(CXX) -o $$@ -c $$< $(CXXFLAGS) $$(DEPFLAGS)))

$(eval $(call BUILD, \
	BENCH_LD, \
	%_bench, \
	%_bench.o bess.a, \
	$(CXX) -o $$@ $$^ $(LDFLAGS) -lbenchmark $(LIBS)))

LIB_OBJS := $(filter-out main.o, $(OBJS))

$(eval $(call BUILD, \
	AR, \
	bess.a, \
	$(LIB_OBJS), \
	$(AR) rcs $$@ $$^))

.PRECIOUS: %.d $(PROTO_HEADERS)

-include $(patsubst %,$(DEPDIR)/%.d,$(basename $(ALL_SRCS)))
