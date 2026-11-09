# Copyright (C) 2026 Lubos Dolezel

project(libpcap)

cmake_minimum_required(VERSION 3.13)

include(InstallSymlink)

# grammar.y and scanner.l are turned into C the way Apple's Xcode build
# rule does it: a pcap_ symbol prefix and a separate scanner header.
BISON_TARGET(pcap_grammar libpcap/grammar.y ${CMAKE_CURRENT_BINARY_DIR}/grammar.c
	COMPILE_FLAGS "--yacc --name-prefix=pcap_"
	DEFINES_FILE ${CMAKE_CURRENT_BINARY_DIR}/grammar.h)
FLEX_TARGET(pcap_scanner libpcap/scanner.l ${CMAKE_CURRENT_BINARY_DIR}/scanner.c
	COMPILE_FLAGS "-Ppcap_ --nounput"
	DEFINES_FILE ${CMAKE_CURRENT_BINARY_DIR}/scanner.h)
ADD_FLEX_BISON_DEPENDENCY(pcap_scanner pcap_grammar)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -nostdinc -w")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -nostdlib")

include_directories(
	${CMAKE_CURRENT_SOURCE_DIR}/libpcap
	${CMAKE_CURRENT_BINARY_DIR}
)

# Same preprocessor settings as Apple's Xcode project
add_definitions(
	-DHAVE_CONFIG_H
	"-D_U_=__attribute__((unused))"
	-Dyylval=pcap_lval
	-DPRIVATE
	-DHAVE_PKTAP_API
)

set(pcap_sources
	libpcap/bpf_dump.c
	libpcap/bpf_filter.c
	libpcap/bpf_image.c
	libpcap/etherent.c
	libpcap/fad-getad.c
	libpcap/fmtutils.c
	libpcap/gencode.c
	libpcap/nametoaddr.c
	libpcap/optimize.c
	libpcap/pcap-bpf.c
	libpcap/pcap-apple-stubs.c
	libpcap/pcap-common.c
	libpcap/pcap-darwin.c
	libpcap/pcap-util.c
	libpcap/pcap.c
	libpcap/pcapng.c
	libpcap/savefile.c
	libpcap/sf-pcap.c
	libpcap/sf-pcapng.c
	${BISON_pcap_grammar_OUTPUTS}
	${FLEX_pcap_scanner_OUTPUTS}
)

set(DYLIB_INSTALL_NAME "/usr/lib/libpcap.A.dylib")
set(DYLIB_COMPAT_VERSION "1.0.0")
set(DYLIB_CURRENT_VERSION "1.0.0")

add_darling_library(pcap SHARED ${pcap_sources})
set_target_properties(pcap PROPERTIES OUTPUT_NAME "pcap.A")
target_link_libraries(pcap system)
# The exported symbols are those of /usr/lib/libpcap.A.dylib on macOS:
# the public pcap API plus Apple's pcap-ng and pktap additions.
target_link_options(pcap PRIVATE
	"-Wl,-exported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/darling/libpcap.exp"
)
make_fat(pcap)

install(TARGETS pcap DESTINATION libexec/darling/usr/lib)
InstallSymlink("libpcap.A.dylib" "${CMAKE_INSTALL_PREFIX}/libexec/darling/usr/lib/libpcap.dylib")

file(GLOB pcap_man3 ${CMAKE_CURRENT_SOURCE_DIR}/libpcap/*.3pcap)
install(FILES ${pcap_man3} DESTINATION libexec/darling/usr/share/man/man3)
