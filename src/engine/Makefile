#!/usr/bin/make -f

NAME ?= stargate-engine

# Compiler flags
CC ?= gcc
CC_ARGS ?= -Wall
OPT_LVL ?= -O2 -g -ggdb3
# These assume a modern x86 CPU, change or remove for other platforms
PLAT_FLAGS ?= -mfpmath=sse -mssse3 -mtune=generic -mstackrealign
LIBEXT ?= so

INC = -Iinclude -Ilibcds/include
SRC = $(shell find src/ libcds/src -type f -name '*.c')

CACHE_LINE_SIZE = $(shell getconf LEVEL1_DCACHE_LINESIZE || echo 64)

OPTIMIZE_FLAGS ?= \
	-flto \
	-fno-stack-protector \
	-fprefetch-loop-arrays \
	# -funroll-loops \

THREAD_LOCAL ?=  -DSG_THREAD_LOCAL=__thread
DEFINES ?= \
	-DCACHE_LINE_SIZE=$(CACHE_LINE_SIZE) \
	-D_GNU_SOURCE \
	-D__USE_GNU \
	$(THREAD_LOCAL)

BASE_FLAGS ?= $(OPTIMIZE_FLAGS) $(DEFINES)

BUILD_FLAGS ?= -std=gnu89 $(PLAT_FLAGS) $(BASE_FLAGS) $(CFLAGS) $(INC) $(SRC)

MATH_LIB_FLAG ?= -lm
ALSA_FLAG ?= $(shell pkg-config --libs alsa)
PORTMIDI_FLAG ?= -lportmidi

PLAT_LINK_CFLAGS ?= $(ALSA_FLAG)

LINK_FLAGS  ?= \
	-lpthread $(PORTMIDI_FLAG) -lportaudio $(LDFLAGS) \
	$(shell pkg-config --libs sndfile fftw3f fftw3) $(MATH_LIB_FLAG) \
	$(PLAT_LINK_CFLAGS)

MINGW_LINK_FLAGS ?= \
	-I/mingw64/include -lpthread $(PORTMIDI_FLAG) -lportaudio $(LDFLAGS) \
	-lsndfile -lfftw3f -lfftw3 $(MATH_LIB_FLAG) -lws2_32 -lDbgHelp


# Debugging flags
# Use -g used for tools other than gdb.
DEBUG_FLAGS ?= -O0 -g -ggdb3
DEBUGGER ?= gdb

# Coverage report flags
GCOVARGS ?= -fprofile-arcs -ftest-coverage  # -fPIC
# Coverage reports will open in this browser
BROWSER ?= firefox

# Install and packaging flags
DESTDIR ?=
BINDIR ?= /usr/bin
INCLUDEDIR ?= /usr/include
ARCH ?= $(shell uname --machine)

# `make perf` counters
PERF_COUNTERS ?= "cache-references,cache-misses,dTLB-loads,\
dTLB-load-misses,iTLB-loads,iTLB-load-misses,L1-dcache-loads,\
L1-dcache-load-misses,L1-icache-loads,L1-icache-load-misses,\
branch-misses,LLC-loads,LLC-load-misses"

# Benchmark flags
# Tells benchmarks that use the ITERATIONS macro how many iterations to attempt
BENCH_ITERATIONS ?= 100000000UL
# Tells the benchmarks to try not using more than this amount of memory.  It
# is not guaranteed that this number will not be exceeded
BENCH_SIZE_MB ?= 500UL

all: engine debug

bench:
	# Run the benchmark, output the results as YAML to stderr
	$(CC) \
	    -fpie $(CC_ARGS) $(BUILD_FLAGS) \
	    $(shell find benchmark -name '*.c') \
	    -DITERATIONS=$(BENCH_ITERATIONS) \
	    -DBENCH_SIZE_MB=$(BENCH_SIZE_MB) \
	    $(LINK_FLAGS) -o $(NAME).benchmark
	./$(NAME).benchmark

bench-test:
	# Run the benchmark through valgrind and gcov
	$(CC) \
	    -fpie $(CC_ARGS) $(DEBUG_FLAGS) $(PLAT_FLAGS) $(GCOVARGS) \
	    $(shell find src benchmark -name '*.c') \
	    -DITERATIONS=100 -DBENCH_SIZE_MB=1 \
	    $(LINK_FLAGS) -o $(NAME).benchmark
	# If Valgrind exits non-zero, try running 'gdb ./engine.tests'
	# to debug the test suite
	valgrind ./$(NAME).benchmark --track-origins=yes
	mkdir html || rm -rf html/*
	gcovr -r . --html --html-details \
		--exclude=tests \
		-o html/coverage.html
	rm -rf *.gcda *.gcno
	$(BROWSER) html/coverage.html &
	# NOTE: You do not need to cover all lines, this is simply for
	# 	debugging purposes

clean:
	# Remove all temporary files
	rm -rf $(NAME)* \
		html/* *.gcda *.gcno *.out *.out.* pahole.txt \
		profile.txt bench.out vgcore* test.wav*

debug:
	# Compile the binary with the appropriate flags to debug in GDB
	$(CC) \
	    -fpie $(CC_ARGS) $(DEBUG_FLAGS) $(BUILD_FLAGS) \
	    $(shell find cli -name '*.c') \
		$(LINK_FLAGS) -o $(NAME)-dbg

debugger:
	make debug
	$(DEBUGGER) ./$(NAME)-dbg

engine:
	# Compile the cli
	$(CC) \
	    -fpie $(CC_ARGS) $(OPT_LVL) $(BUILD_FLAGS) \
	    $(shell find cli -name '*.c') $(LINK_FLAGS) -o $(NAME)

gprof:
	# Profile which functions the test suite spends the most time
	# in using gprof
	$(CC) -fpie $(CC_ARGS) $(BUILD_FLAGS) -pg \
	    $(shell find benchmark -name *.c) \
	    $(LINK_FLAGS) -o $(NAME).gprof
	./$(NAME).gprof
	gprof ./$(NAME).gprof > profile.txt
	less profile.txt

install:
	# Install the binary
	install -d $(DESTDIR)/$(BINDIR)
	install $(NAME) $(DESTDIR)/$(BINDIR)/

install-devel:
	# Install the headers
	install -d $(DESTDIR)
	cp -r include $(DESTDIR)

lines-of-code:
	# source code
	find src -name '*.c' -exec cat {} \; | wc -l
	# tests
	find tests -name '*.c' -exec cat {} \; | wc -l
	# headers
	find include -name '*.h' -exec cat {} \; | wc -l
	# benchmarks
	find benchmark -name '*.c' -exec cat {} \; | wc -l

mac_osx:
	CC=clang make engine

mingw:
	# Build the Windows engine using MSYS2
	rm -f stargate-engine.pdb
	LINK_FLAGS="$(MINGW_LINK_FLAGS)" make engine
	make -C .. commit_hash
	###################
	#### IMPORTANT ####
	###################
	# Go run the following command in the
	# Visual Studio Developer Command Prompt
	# from the engine directory:
	# .\cv2pdb\cv2pdb.exe .\stargate-engine.exe

pahole:
	# Check struct alignnment using pahole
	$(CC) \
		-fpie $(CC_ARGS) $(DEBUG_FLAGS) $(BUILD_FLAGS) \
	    $(shell find tests -name '*.c') \
	    $(LINK_FLAGS) -o $(NAME).pahole
	pahole $(NAME).pahole > pahole.txt
	less pahole.txt

perf:
	# Profile system performance counters using perf
	$(CC) \
		-fpie $(CC_ARGS) $(BUILD_FLAGS) \
	    $(shell find benchmark -name '*.c') \
	    $(LINK_FLAGS) -o $(NAME).perf
	perf stat -e $(PERF_COUNTERS) ./$(NAME).perf

test:
	# Compile and run the test suite through Valgrind to check for
	# memory errors, then generate an HTML code coverage report
	# using gcovr
	$(CC) \
		-fpie $(CC_ARGS) $(BUILD_FLAGS) $(DEBUG_FLAGS) $(PLAT_FLAGS) \
		$(GCOVARGS) -fstack-protector \
	    $(shell find tests -name '*.c') \
	    -Itests $(LINK_FLAGS) -o $(NAME).tests
	rm -rf tmp
	mkdir -p tmp
	# If Valgrind exits non-zero, try running 'gdb ./engine.tests'
	# to debug the test suite
	valgrind ./$(NAME).tests --track-origins=yes
	mkdir html || rm -rf html/*
	gcovr -r . --html --html-details \
		--exclude=benchmark \
		--exclude=libcds \
		-o html/coverage.html
	rm -rf *.gcda *.gcno
	$(BROWSER) html/coverage.html &

