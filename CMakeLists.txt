cmake_minimum_required(VERSION 3.6)
project(musl LANGUAGES C CXX ASM)

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)
include(CheckCSourceCompiles)

function(check_linker_flag flag out_var)
	cmake_push_check_state()
	set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} ${flag}")
	check_cxx_compiler_flag("" ${out_var})
	cmake_pop_check_state()
endfunction()

function(check_preprocessor_flag flag out_var)
	set(MUSL_HAS_PREPROCESSOR_${flag} No)
	check_c_source_compiles(
		"
		typedef int x;
		#if ${flag}
		#error yes
		#endif
		"
		MUSL_HAS_PREPROCESSOR_${flag}
	)

	if(${MUSL_HAS_PREPROCESSOR_${flag}})
		set(out_var No)
	else()
		set(out_var Yes)
	endif()
endfunction()

set(MUSL_TOOLS "" CACHE INTERNAL "Musl Toolchain Wrapper")
set(MUSL_TOOLS_LIBS ""  CACHE INTERNAL "Musl Toolchain Wrapper Libraries")
set(MUSL_ARCH "" CACHE INTERNAL "Musl target arch")

set(MUSL_WITH_MALLOC	"mallocng"		CACHE STRING	"Malloc Implementation")
set(MUSL_LGCC			""				CACHE STRING	"Compiler Runtime Library")

if(NOT EXISTS ${CMAKE_CURRENT_LIST_DIR}/src/malloc/${MUSL_WITH_MALLOC})
	message(FATAL_ERROR "error: chosen malloc implementation ${MUSL_WITH_MALLOC} does not exist")
endif()

#
# Figure out options to force errors on unknown flags.
#
check_c_compiler_flag(-Werror=unknown-warning-option			MUSL_HAS_UNKNOWN_WARNING)
check_c_compiler_flag(-Werror=unused-command-line-argument		MUSL_HAS_UNUSED_ARGUMENT)
check_c_compiler_flag(-Werror=ignored-optimization-argument		MUSL_HAS_IGNORED_OPTIMIZATION)
check_linker_flag(-Werror=unknown-warning-option				MUSL_HAS_LINKER_UNKNOWN_WARNING)
check_linker_flag(-Werror=unused-command-line-argument			MUSL_HAS_LINKER_UNUSED_ARGUMENT)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	list(APPEND MUSL_TOOLS
		${CMAKE_CURRENT_LIST_DIR}/obj/musl-clang
		${CMAKE_CURRENT_LIST_DIR}/obj/ld.musl-clang
	)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	list(APPEND MUSL_TOOLS ${CMAKE_CURRENT_LIST_DIR}/obj/musl-gcc)
	list(APPEND MUSL_TOOLS_LIBS ${CMAKE_CURRENT_LIST_DIR}/lib/musl-gcc.specs)
else()
	message(FATAL_ERROR "error: could not find an appropriate toolchain wrapper")
endif()

if(${CMAKE_C_COMPILER_TARGET} MATCHES "^arm.*")
	set(MUSL_ARCH "arm")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^aarch64.*")
	set(MUSL_ARCH "aarch64")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^i?86-nt32.*")
	set(MUSL_ARCH "nt32")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^i?86.*")
	set(MUSL_ARCH "i386")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^(x86_64-x32.*|x32.*|x86_64.*x32)")
	set(MUSL_ARCH "x32")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^x86_64-nt64.*")
	set(MUSL_ARCH "nt64")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^x86_64.*")
	set(MUSL_ARCH "x86_64")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^m68k.*")
	set(MUSL_ARCH "m68k")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^(mips64.*|mipsisa64.*)")
	set(MUSL_ARCH "mips64")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^mips.*")
	set(MUSL_ARCH "mips")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^microblaze.*")
	set(MUSL_ARCH "microblaze")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^or1k.*")
	set(MUSL_ARCH "or1k")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^(powerpc64.*|ppc64.*)")
	set(MUSL_ARCH "powerpc64")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^(powerpc.*|ppc.*)")
	set(MUSL_ARCH "powerpc")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^riscv64.*")
	set(MUSL_ARCH "riscv64")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^(sh[1-9bel-].*|sh|superh.*)")
	set(MUSL_ARCH "sh")
elseif(${CMAKE_C_COMPILER_TARGET} MATCHES "^s390x.*")
	set(MUSL_ARCH "s390x")
else()
	message(FATAL_ERROR "error: unknown or unsupported target ${CMAKE_C_COMPILER_TARGET}")
endif()

#
# Try to get a conforming C99 freestanding environment
#
check_c_compiler_flag(-std=c99								MUSL_HAS_C99)
check_c_compiler_flag(-nostdinc								MUSL_HAS_NOSTDINC)
check_c_compiler_flag(-ffreestanding						MUSL_HAS_FREESTANDING)
check_c_compiler_flag(-fno-builtin							MUSL_HAS_NO_BUILTIN)
check_c_compiler_flag(-fexcess-precision=standard			MUSL_HAS_EXCESS_PRECISION)

if(${MUSL_ARCH} STREQUAL "i386")
	check_c_compiler_flag(-ffloat-store						MUSL_HAS_FLOAT_STORE)
endif()

check_c_compiler_flag(-frounding-math						MUSL_HAS_ROUNDING_MATH)

#
# We may use the may_alias attribute if __GNUC__ is defined, so
# if the compiler defines __GNUC__ but does not provide it,
# it must be defined away as part of the CFLAGS.
#
check_c_source_compiles(
	"
		typedef int
		#ifdef __GNUC__
		__attribute__((__may_alias__))
		#endif
		x;
	"
	MUSL_HAS_MAY_ALIAS
)

#
# The GNU toolchain defaults to assuming unmarked files need an
# executable stack, potentially exposing vulnerabilities in programs
# linked with such object files. Fix this.
#
check_c_compiler_flag(-Wa,--noexecstack 					MUSL_HAS_NOEXECSTACK)

#
# Check for options to disable stack protector, which needs to be
# disabled for a few early-bootstrap translation units. If not found,
# this is not an error; we assume the toolchain does not do ssp.
#
check_c_compiler_flag(-fno-stack-protector					MUSL_HAS_NO_STACK_PROTECTOR)

#
# Check for options that may be needed to prevent the compiler from
# generating self-referential versions of memcpy,, memmove, memcmp,
# and memset. Really, we should add a check to determine if this
# option is sufficient, and if not, add a macro to cripple these
# functions with volatile...
#
check_c_compiler_flag(-fno-tree-loop-distribute-patterns	MUSL_HAS_NO_TREE_LOOP)

# Always try -pipe
check_c_compiler_flag(-pipe									MUSL_HAS_PIPE)

# Omit frame pointer
check_c_compiler_flag(-fomit-frame-pointer					MUSL_HAS_OMIT_FRAME)

#
# Modern GCC wants to put DWARF tables (used for debugging and
# unwinding) in the loaded part of the program where they are
# unstrippable. These options force them back to debug sections (and
# cause them not to get generated at all if debugging is off).
#
check_c_compiler_flag(-fno-unwind-tables					MUSL_HAS_NO_UNWIND_TABLES)
check_c_compiler_flag(-fno-asynchronous-unwind-tables		MUSL_HAS_NO_ASYNC_UNWIND_TABLES)

#
# Attempt to put each function and each data object in its own
# section. This both allows additional size optimizations at link
# time and works around a dangerous class of compiler/assembler bugs
# whereby relative address expressions are constant-folded by the
# assembler even when one or more of the symbols involved is
# replaceable. See gas pr 18561 and gcc pr 66609, 68178, etc.
#
check_c_compiler_flag(-ffunction-sections					MUSL_HAS_FUNCTION_SECTIONS)
check_c_compiler_flag(-fdata-sections						MUSL_HAS_DATA_SECTIONS)

#
# On x86, make sure we don't have incompatible instruction set
# extensions enabled by default. This is bad for making static binaries.
# We cheat and use i486 rather than i386 because i386 really does not
# work anyway (issues with atomic ops).
# Some build environments pass -march and -mtune options via CC, so
# check both CC and CFLAGS.
#
if(${MUSL_ARCH} STREQUAL "i386")
	if(NOT "${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE}}" MATCHES "-march=")
		check_c_compiler_flag(-march=i486					MUSL_HAS_MARCH)
	endif()

	if(NOT "${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE}}" MATCHES "-mtune=")
		check_c_compiler_flag(-mtune=generic				MUSL_HAS_MTUNE)
	endif()
endif()

#
# GCC defines -w as overriding any -W options, regardless of order, but
# clang has a bunch of annoying warnings enabled by default and needs -w
# to start from a clean slate. So use -w if building with clang. Also
# turn off a common on-by-default cast warning regardless of compiler.
#
if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	check_c_compiler_flag(-w								MUSL_HAS_W)
endif()

check_c_compiler_flag(-Wno-pointer-to-int-cast				MUSL_HAS_NO_POINTER_TO_INT)

#
# Even with -std=c99, gcc accepts some constructs which are constraint
# violations. We want to treat these as errors regardless of whether
# other purely stylistic warnings are enabled -- especially implicit
# function declarations, which are a dangerous programming error.
#
check_c_compiler_flag(-Werror=implicit-function-declaration	MUSL_HAS_IMPLICIT_FUNCTION)
check_c_compiler_flag(-Werror=implicit-int					MUSL_HAS_IMPLICIT_INT)
check_c_compiler_flag(-Werror=pointer-sign					MUSL_HAS_POINTER_SIGN)
check_c_compiler_flag(-Werror=pointer-arith					MUSL_HAS_POINTER_ARITH)
check_c_compiler_flag(-Werror=int-conversion				MUSL_HAS_INT_CONVERSION)
check_c_compiler_flag(-Werror=incompatible-pointer-types	MUSL_HAS_INCOMPATIBLE_POINTER)
check_c_compiler_flag(-Werror=discarded-qualifiers			MUSL_HAS_DISCARD_QUALIFIERS)
check_c_compiler_flag(-Werror=discarded-array-qualifiers	MUSL_HAS_DISCARD_ARRAY_QUALIFIERS)

#
# GCC ignores unused arguements by default, but Clang needs this extra
# parameter to stop printing warnings about LDFLAGS passed during
# compiling stage and CFLAGS passed during linking stage.
#
if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	check_c_compiler_flag(-Qunused-arguments				MUSL_HAS_QUNUSED_ARGUMENTS)
endif()

check_c_compiler_flag(-Waddress								MUSL_HAS_WADDRESS)
check_c_compiler_flag(-Warray-bounds						MUSL_HAS_WARRAY_BOUNDS)
check_c_compiler_flag(-Wchar-subscripts						MUSL_HAS_WCHAR_SUBSCRIPTS)
check_c_compiler_flag(-Wduplicate-decl-specifier			MUSL_HAS_WDUPLICATE_DECL)
check_c_compiler_flag(-Winit-self							MUSL_HAS_WINIT_SELF)
check_c_compiler_flag(-Wreturn-type							MUSL_HAS_WRETURN_TYPE)
check_c_compiler_flag(-Wsequence-point						MUSL_HAS_WSEQUENCE_POINT)
check_c_compiler_flag(-Wstrict-aliasing						MUSL_HAS_WSTRICT_ALIASING)
check_c_compiler_flag(-Wunused-function						MUSL_HAS_WUNUSED_FUNCTION)
check_c_compiler_flag(-Wunused-label						MUSL_HAS_WUNUSED_LABEL)
check_c_compiler_flag(-Wunused-variable						MUSL_HAS_WUNUSED_VARIABLE)

# Reduce space lost to padding for alignment purposes by sorting data
# objects according to their alignment reqirements. This approximates
# optimal packing.
check_linker_flag(-Wl,--sort-section,alignment				MUSL_HAS_SORT_SECTION)
check_linker_flag(-Wl,--sort-common							MUSL_HAS_SORT_COMMON)

# When linking shared library, drop dummy weak definitions that were
# replaced by strong definitions from other translation units.
check_linker_flag(-Wl,--gc-sections							MUSL_HAS_GC_SECTIONS)

# Some patched GCC builds have these defaults messed up...
check_linker_flag(-Wl,--hash-style=both						MUSL_HAS_HASH_STYLE)

# Prevent linking if there are undefined symbols; if any exist,
# libc.so will crash at runtime during relocation processing.
# The common way this can happen is failure to link the compiler
# runtime library; implementation error is also a possibility.
check_linker_flag(-Wl,--no-undefined						MUSL_HAS_NO_UNDEFINED)

# Avoid exporting symbols from compiler runtime libraries. They
# should be hidden anyway, but some toolchains including old gcc
# versions built without shared library support and pcc are broken.
check_linker_flag(-Wl,--exclude-libs=ALL					MUSL_HAS_EXCLUDE_LIBS)

# Public data symbols must be interposable to allow for copy
# relocations, but otherwise we want to bind symbols at libc link
# time to eliminate startup relocations and PLT overhead. Use
# --dynamic-list rather than -Bsymbolic-functions for greater
# control over what symbols are left unbound.
set(MUSL_DYNAMIC_LIST ${CMAKE_CURRENT_LIST_DIR}/dynamic.list)
check_linker_flag(-Wl,--dynamic-list="${MUSL_DYNAMIC_LIST}"	MUSL_HAS_DYNAMIC_LIST)

# Find compiler runtime library
if(NOT MUSL_LGCC)
	check_linker_flag("-lgcc"								MUSL_LGCC)

	if(NOT MUSL_LGCC)
		check_linker_flag("-lgcc_eh"						MUSL_LGCC)
	endif()

	if(NOT MUSL_LGCC)
		set(libgcc_cmd
			${CMAKE_C_COMPILER}
			${CMAKE_C_FLAGS}
			${CMAKE_LINKER_FLAGS}
			-print-libgcc-file-name
		)

		execute_process(
			COMMAND "${libgcc_cmd}"
			RESULT_VARIABLE libgcc_check_ret
			OUTPUT_VARIABLE libgcc_path
			OUTPUT_STRIP_TRAILING_WHITESPACE
		)

		if (${libgcc_check_ret} EQUAL 0)
			set(MUSL_LGCC "${libgcc_path}")
		endif()
	endif()

	if(NOT MUSL_LGCC)
		set(libgcc_cmd
			${CMAKE_C_COMPILER}
			${CMAKE_C_FLAGS}
			${CMAKE_LINKER_FLAGS}
			-print-file-name=libpcc.a
		)

		execute_process(
			COMMAND "${libgcc_cmd}"
			RESULT_VARIABLE libgcc_check_ret
			OUTPUT_VARIABLE libgcc_path
			OUTPUT_STRIP_TRAILING_WHITESPACE
		)

		if (${libgcc_check_ret} EQUAL 0)
			set(MUSL_LGCC "${libgcc_path}")
		endif()
	endif()

	if(NOT MUSL_LGCC)
		message(FATAL_ERROR "error: unable to determine compiler runtime library")
	endif()
endif()

if(${MUSL_ARCH} STREQUAL "i386")
	check_c_source_compiles(
		"
		int foo(int x) { __asm__ ( \"\" : \"+b\"(x) ); return x; }
		"
		MUSL_HAS_PIC_ASM
	)
endif()

if(${MUSL_ARCH} STREQUAL "x86_64")
	check_preprocessor_flag("__ILP32__"						musl_ilp)

	if(musl_ilp)
		set(MUSL_ARCH "x32")
	endif()
endif()

if(${MUSL_ARCH} STREQUAL "arm")
	check_preprocessor_flag("__thumb2__"					musl_thumb2)

	if(musl_thumb2)
		check_c_compiler_flag("-mimplicit-it=always"		MUSL_HAS_IMPLICIT_IT)
		check_c_compiler_flag("-Wa,-mimplicit-it=always"	MUSL_HAS_WA_IMPLICIT_IT)
		check_c_compiler_flag("-Wa,-mthumb"					MUSL_HAS_WA_THUMB)
	endif()

	check_preprocessor_flag("__ARMEB__"						musl_eb)

	if(musl_eb)
		set(MUSL_SUBARCH	"eb")
	endif()

	check_preprocessor_flag("__ARM_PCS_VFP"					musl_hf)

	if(musl_hf)
		set(MUSL_SUBARCH	"hf")
	endif()

	# Versions of clang up until at least 3.8 have the wrong constraint codes
	# for floating point operands to inline asm. Detect this so the affected
	# source files can just disable the asm.
	if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
		check_c_source_compiles(
			"
			float f(float x) { __asm__(\"\":\"+t\"(x)); return x; }
			"
			MUSL_HAS_VFP_CONSTRAINTS
		)
	endif()
endif()

if(${MUSL_ARCH} STREQUAL "aarch64")
	check_preprocessor_flag("__AARCH64EB__"		musl_be)

	if(musl_be)
		set(MUSL_SUBARCH	"_be")
	endif()
endif()

#
# Some archs (powerpc) have different possible long double formats
# that the compiler can be configured for. The logic for whether this
# is supported is in bits/float.h; in general, it is not. We need to
# check for mismatches here or code in printf, strotd, and scanf will
# be dangerously incorrect because it depends on (1) the macros being
# correct, and (2) IEEE semantics.
#
check_c_source_compiles(
	"
	#include <float.h>
	#define C(m,s) (m==LDBL_MANT_DIG && s==sizeof(long double))
	typedef char ldcheck[(C(53,8)||C(64,12)||C(64,16)||C(113,16))*2-1];
	"
	musl_long_double
)

if(NOT musl_long_double)
	message(FATAL_ERROR "error: unsupported long double type")
endif()

#
# Some build systems globally pass in broken CFLAGS like -ffast-math
# for all packages. On recent GCC we can detect this and error out
# early rather than producing a seriously-broken math library.
#
check_preprocessor_flag("__FAST_MATH__"		musl_fast_math)

if(musl_fast_math)
	message(FATAL_ERROR "error: compiler has broken floating point; check CFLAGS")
endif()

set(MUSL_EMPTY_LIBS
	${CMAKE_CURRENT_BINARY_DIR}/lib/libm.a
	${CMAKE_CURRENT_BINARY_DIR}/lib/librt.a
	${CMAKE_CURRENT_BINARY_DIR}/lib/libpthread.a
	${CMAKE_CURRENT_BINARY_DIR}/lib/libcrypt.a
	${CMAKE_CURRENT_BINARY_DIR}/lib/libutil.a
	${CMAKE_CURRENT_BINARY_DIR}/lib/libxnet.a
	${CMAKE_CURRENT_BINARY_DIR}/lib/libresolv.a
	${CMAKE_CURRENT_BINARY_DIR}/lib/libdl.a
)

add_custom_target(musl_empty_libs_target
	DEPENDS
		${MUSL_EMPTY_LIBS}
)

foreach(lib ${MUSL_EMPTY_LIBS})
	add_custom_command(#${lib}
		OUTPUT ${lib}
		COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/lib
		COMMAND rm -f ${lib}
		COMMAND ${CMAKE_AR} rc ${lib}
	)

	install(
		FILES
			${lib}
		COMPONENT musl-libs
		DESTINATION musl/lib
	)
endforeach()

set(MUSL_SRC_DIRS
	${CMAKE_CURRENT_SOURCE_DIR}/src/malloc/${MUSL_WITH_MALLOC}
	${CMAKE_CURRENT_SOURCE_DIR}/crt
	${CMAKE_CURRENT_SOURCE_DIR}/ldso
)

file(GLOB children ${CMAKE_CURRENT_SOURCE_DIR}/src/*)

foreach(child ${children})
	if(IS_DIRECTORY ${child})
		list(APPEND MUSL_SRC_DIRS ${child})
	endif()
endforeach()

set(MUSL_SRCS "")

foreach(dir ${MUSL_SRC_DIRS})
	file(GLOB srcs ${dir}/*.c)
	list(APPEND MUSL_SRCS ${srcs})
endforeach()

add_custom_target(musl_bits_target
	DEPENDS
		${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/alltypes.h
		${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/syscall.h
		${CMAKE_CURRENT_BINARY_DIR}/obj/src/internal/version.h
)

add_custom_command(#bits_alltypes
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/alltypes.h
	COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits
	COMMAND sed -f ${CMAKE_CURRENT_SOURCE_DIR}/tools/mkalltypes.sed
		${CMAKE_CURRENT_SOURCE_DIR}/arch/${MUSL_ARCH}/bits/alltypes.h.in
		${CMAKE_CURRENT_SOURCE_DIR}/include/alltypes.h.in >
		${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/alltypes.h
)

add_custom_command(#bits_syscall
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/syscall.h
	COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits
	COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/arch/${MUSL_ARCH}/bits/syscall.h.in
		${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/syscall.h
	COMMAND sed -n -e s/__NR_/SYS_/p <
		${CMAKE_CURRENT_SOURCE_DIR}/arch/${MUSL_ARCH}/bits/syscall.h.in >>
		${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/syscall.h
)

add_custom_command(#version_header
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/obj/src/internal/version.h
	COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/obj/src/internal
	COMMAND printf
		"#define VERSION \"%s\"\\n" "\$\$(cd ${CMAKE_CURRENT_SOURCE_DIR}; sh tools/version.sh)" >
		${CMAKE_CURRENT_BINARY_DIR}/obj/src/internal/version.h
	VERBATIM
)

add_library(c STATIC ${MUSL_SRCS})

target_compile_definitions(c
	PUBLIC
		-D_XOPEN_SOURCE=700
)

target_compile_options(c
	PUBLIC
		-Wno-unused-parameter
		-Wno-shift-op-parentheses
		-Wno-sign-compare
		-Wno-empty-translation-unit
		-Wno-logical-op-parentheses
		-Wno-bitwise-op-parentheses
		-Wno-macro-redefined
		-Wno-missing-braces
)

target_include_directories(c
	PUBLIC
		${CMAKE_CURRENT_SOURCE_DIR}/arch/${MUSL_ARCH}
		${CMAKE_CURRENT_SOURCE_DIR}/arch/generic
		${CMAKE_CURRENT_BINARY_DIR}/obj/src/internal
		${CMAKE_CURRENT_SOURCE_DIR}/src/include
		${CMAKE_CURRENT_SOURCE_DIR}/src/internal
		${CMAKE_CURRENT_BINARY_DIR}/obj/include
		${CMAKE_CURRENT_SOURCE_DIR}/include
)

install(
	TARGETS c
	COMPONENT musl-libs
	DESTINATION musl/lib
)

set(MUSL_GENH
	${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/alltypes.h
	${CMAKE_CURRENT_BINARY_DIR}/obj/include/bits/syscall.h
)

file(GLOB MUSL_ARCH_INCLUDES ${CMAKE_CURRENT_SOURCE_DIR}/arch/${MUSL_ARCH}/bits/*.h)
file(GLOB MUSL_GENERIC_INCLUDES ${CMAKE_CURRENT_SOURCE_DIR}/arch/generic/bits/*.h)

install(
	DIRECTORY
		${CMAKE_CURRENT_SOURCE_DIR}/include
	COMPONENT musl-headers
	DESTINATION musl
	FILES_MATCHING PATTERN "*.h"
)

install(
	FILES
		${MUSL_GENH}
		${MUSL_ARCH_INCLUDES}
		${MUSL_GENERIC_INCLUDES}
	COMPONENT musl-bits-headers
	DESTINATION musl/include/bits
)

set(MUSL_CRT_OBJS
	${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/c.dir/crt/crt1.c.o
	${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/c.dir/crt/crti.c.o
	${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/c.dir/crt/crtn.c.o
	${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/c.dir/crt/rcrt1.c.o
	${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/c.dir/crt/Scrt1.c.o
)

foreach(obj ${MUSL_CRT_OBJS})
	get_filename_component(obj_wle ${obj} NAME_WLE)

	install(
		FILES
			${obj}
		RENAME ${obj_wle}.o
		COMPONENT musl-crt-objs
		DESTINATION musl/lib
	)
endforeach()

add_custom_target(musl_bin_target
	DEPENDS
		${CMAKE_CURRENT_BINARY_DIR}/obj/ld.musl-clang
		${CMAKE_CURRENT_BINARY_DIR}/obj/musl-clang
)

add_custom_command(
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/obj/ld.musl-clang
	COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/obj
	COMMAND sed -e 's!@CC@!clang!g'
		-e 's!@PREFIX@!${CMAKE_INSTALL_PREFIX}/musl!g'
		-e 's!@INCDIR@!${CMAKE_INSTALL_PREFIX}/musl/include!g'
		-e 's!@LIBDIR@!${CMAKE_INSTALL_PREFIX}/musl/lib!g'
		-e 's!@LDSO@!${CMAKE_INSTALL_PREFIX}/musl/lib!g'
		${CMAKE_CURRENT_SOURCE_DIR}/tools/ld.musl-clang.in >
		${CMAKE_CURRENT_BINARY_DIR}/obj/ld.musl-clang
)

add_custom_command(
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/obj/musl-clang
	COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/obj
	COMMAND sed -e 's!@CC@!clang!g'
		-e 's!@PREFIX@!${CMAKE_INSTALL_PREFIX}/musl!g'
		-e 's!@INCDIR@!${CMAKE_INSTALL_PREFIX}/musl/include!g'
		-e 's!@LIBDIR@!${CMAKE_INSTALL_PREFIX}/musl/lib!g'
		-e 's!@LDSO@!${CMAKE_INSTALL_PREFIX}/musl/lib!g'
		${CMAKE_CURRENT_SOURCE_DIR}/tools/musl-clang.in >
		${CMAKE_CURRENT_BINARY_DIR}/obj/musl-clang
)

install(
	FILES
		${CMAKE_CURRENT_BINARY_DIR}/obj/ld.musl-clang
		${CMAKE_CURRENT_BINARY_DIR}/obj/musl-clang
	COMPONENT musl-bin
	PERMISSIONS
		OWNER_READ OWNER_WRITE OWNER_EXECUTE
		GROUP_READ GROUP_EXECUTE
		WORLD_READ WORLD_EXECUTE
	DESTINATION musl/bin
)

add_dependencies(c musl_bits_target musl_empty_libs_target musl_bin_target)
