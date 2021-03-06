##
## CMake build script for the yaramod library.
##

# Source files.
set(SOURCES
	builder/yara_expression_builder.cpp
	builder/yara_file_builder.cpp
	builder/yara_hex_string_builder.cpp
	builder/yara_rule_builder.cpp
	parser/parser_driver.cpp
	types/hex_string.cpp
	types/literal.cpp
	types/meta.cpp
	types/modules/androguard_module.cpp
	types/modules/cuckoo_module.cpp
	types/modules/dex_module.cpp
	types/modules/dotnet_module.cpp
	types/modules/elf_module.cpp
	types/modules/hash_module.cpp
	types/modules/macho_module.cpp
	types/modules/magic_module.cpp
	types/modules/math_module.cpp
	types/modules/module.cpp
	types/modules/pe_module.cpp
	types/modules/time_module.cpp
	types/plain_string.cpp
	types/rule.cpp
	types/yara_file.cpp
	utils/filesystem.cpp
	utils/utils.cpp
	yaramod.cpp
)

if(NOT TARGET yaramod)
	# Setup automatic generation of lexer and parser source files.
	file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/yaramod/yy")
	FLEX_TARGET(YaraLexer parser/yy/lexer.l "${CMAKE_CURRENT_BINARY_DIR}/yaramod/yy/yy_lexer.cpp")
	BISON_TARGET(YaraParser parser/yy/parser.y "${CMAKE_CURRENT_BINARY_DIR}/yaramod/yy/yy_parser.cpp")
	list(APPEND SOURCES "${FLEX_YaraLexer_OUTPUTS}")
	list(APPEND SOURCES "${BISON_YaraParser_OUTPUTS}")

	# Turn off warnings for files generated by Flex and Bison.
	if(MSVC)
		set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/yaramod/yy/yy_lexer.cpp PROPERTIES COMPILE_FLAGS "/W0")
		set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/yaramod/yy/yy_parser.cpp PROPERTIES COMPILE_FLAGS "/W0")
	else() # Linux/MSYS2
		set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/yaramod/yy/yy_lexer.cpp PROPERTIES COMPILE_FLAGS "-w")
		set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/yaramod/yy/yy_parser.cpp PROPERTIES COMPILE_FLAGS "-w")

		if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
			# Define _Noreturn to prevent Clang from emitting the following warning:
			#
			#   yy_parser.hpp:101:11:
			#     warning: macro name is a reserved identifier [-Wreserved-id-macro]
			#   #define _Noreturn YY_ATTRIBUTE ((__noreturn__))
			#
			# _Noreturn is a C keyword (since C11) but we are using C++, not C. It
			# is not even used anywhere in the generated files. I tried to specify
			# --language=c++ when running bison but it did not have any effect.
			add_definitions(-D_Noreturn)
		endif()
	endif()

	# Library.
	add_library(yaramod ${SOURCES})

	# Library includes and interface.
	target_include_directories(yaramod PRIVATE "${YARAMOD_INCLUDE_DIR}")
	target_include_directories(yaramod SYSTEM INTERFACE "${YARAMOD_INCLUDE_DIR}")
	# We need to be able to include the generated files from the build directory.
	# By declaring the include path as a system path, we automatically discard
	# warnings from the generated header files, which is useful.
	target_include_directories(yaramod SYSTEM PUBLIC "${CMAKE_CURRENT_BINARY_DIR}")
	# To ensure that the compiler is able to find <FlexLexer.h> (this is needed at
	# least for MSVC 2015):
	target_include_directories(yaramod SYSTEM PUBLIC "${FLEX_INCLUDE_DIRS}")
	target_include_directories(yaramod SYSTEM PUBLIC "${YARAMOD_DEPS_DIR}")

	# Platform dependant defines.
	if(WIN32)
		target_compile_definitions(yaramod PRIVATE -DYARAMOD_OS_WINDOWS)
	elseif(UNIX)
		target_compile_definitions(yaramod PRIVATE -DYARAMOD_OS_LINUX)
	endif()

	# Linking.
	if(WIN32)
		target_link_libraries(yaramod shlwapi)
	endif()

	# Python module.
	if(YARAMOD_PYTHON)
		add_subdirectory(python)
	endif()

	# Examples.
	if(YARAMOD_EXAMPLES)
		add_subdirectory(examples)
	endif()

	# Installation.
	install(
		TARGETS yaramod
		ARCHIVE DESTINATION "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}"
		LIBRARY DESTINATION "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}"
	)
	install(
		DIRECTORY "${YARAMOD_INCLUDE_DIR}/"
		DESTINATION "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR}"
	)
	install(
		DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/yaramod/"
		DESTINATION "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR}/yaramod"
		FILES_MATCHING
			PATTERN "*.hpp"
			PATTERN "*.hh"
	)
	install(
		DIRECTORY "${YARAMOD_DEPS_DIR}/optional_lite"
		DESTINATION "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR}"
	)
	install(
		DIRECTORY "${YARAMOD_DEPS_DIR}/variant"
		DESTINATION "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR}"
	)
endif()
