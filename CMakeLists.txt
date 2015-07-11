#
# Copyright (c) 2015, Luca Fulchir<luca@fulchir.it>, All rights reserved.
#
# This file is part of "libRaptorQ".
#
# libRaptorQ is free software: you can redistribute it and/or modify
# it under the terms of the GNU Lesser General Public License as
# published by the Free Software Foundation, either version 3
# of the License, or (at your option) any later version.
#
# libRaptorQ is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# and a copy of the GNU Lesser General Public License
# along with libRaptorQ.  If not, see <http://www.gnu.org/licenses/>.

PROJECT(libRaptorQ)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
AUX_SOURCE_DIRECTORY(. SRC_LIST)
INCLUDE(flags OPTIONAL)

IF(NOT CMAKE_BUILD_TYPE) # if not specified, default to "Release"
  SET(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build mode") # default build type
  SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS Debug Release RelWithDebInfo MinSizeRel)
ENDIF()

MESSAGE(STATUS "Build selected: ${CMAKE_BUILD_TYPE}")

IF(NOT CMAKE_CXX_COMPILER)
  SET(CMAKE_CXX_COMPILER "g++" CACHE STRING "C++ Compiler") # default compiler
ENDIF()
IF(NOT CMAKE_C_COMPILER)
  SET(CMAKE_C_COMPILER "gcc" CACHE STRING "C Compiler") # default compiler
ENDIF()
OPTION(OVERRIDE_CMAKE_GENERATOR "override cmake generator compiler" ON)
IF(OVERRIDE_CMAKE_GENERATOR)
  IF(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR ${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
    SET_PROPERTY(CACHE CMAKE_CXX_COMPILER PROPERTY STRINGS clang++ g++ c++)
    SET_PROPERTY(CACHE CMAKE_C_COMPILER PROPERTY STRINGS clang gcc cc)
    MARK_AS_ADVANCED(CLEAR CMAKE_CXX_COMPILER CMAKE_C_COMPILER)
  ENDIF()
ENDIF()
SET(STDLIB "libstdc++" CACHE STRING "Standard library to use")
SET_PROPERTY(CACHE STDLIB PROPERTY STRINGS libc++ libstdc++)

# Link time optimization: smaller, better optimized libraries
OPTION(LTO "Link Time Optimization" ON)

# profiling support: optimize the library
OPTION(PROFILING "Optimize library assembly" ON)

OPTION(BUILD_SHARED_LIB "Build and install Shared library" ON)

SET(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake; ${CMAKE_MODULE_PATH})

FIND_PACKAGE(Threads REQUIRED)
FIND_PACKAGE(git)
FIND_PACKAGE(eigen REQUIRED)
INCLUDE_DIRECTORIES(SYSTEM ${EIGEN3_INCLUDE_DIR})

#This only works if *FIRST* you checkout a commit and *THEN* you run cmake
IF(GIT_FOUND)
  EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} rev-parse --sq HEAD OUTPUT_VARIABLE SEED ERROR_VARIABLE gitErr)
  IF(gitErr)
    SET(SEED "RaptorQ_giterr")
    MESSAGE(WARNING "git rev-parse failed. compiler seed is now constant.")
  ENDIF()
ELSE()
  SET(SEED "RaptorQ_nongit")
  MESSAGE(WARNING "Not inside a git repository. compiler seed is now constant.")
ENDIF()
SET(DETERMINISTIC "-frandom-seed=${SEED}")

IF(LTO)
	MESSAGE(STATUS "Link Time Optimization activated")
ELSE()
	MESSAGE(STATUS "Link Time Optimization deactivated")
ENDIF()
#activate profiling if requested.
# profiling needs to build a test, which needs threading.
IF (PROFILING)
	MESSAGE(STATUS "Profiling activated")
ELSE()
	MESSAGE(STATUS "Profiling deactivated")
ENDIF()

# setup the compiler and the standard library
IF(STDLIB AND OVERRIDE_CMAKE_GENERATOR)
	IF(${STDLIB} STREQUAL "libc++" AND NOT (${CMAKE_CXX_COMPILER} MATCHES "^.*c[+][+]$" OR ${CMAKE_CXX_COMPILER} MATCHES "^(.*/)g[+][+]$"))
		# gcc is particoularly unhappy with libc++ headers...
		INCLUDE_DIRECTORIES(SYSTEM ${CLANG_C_INCLUDE_DIR})
		FIND_PACKAGE(std_clang REQUIRED)
		INCLUDE_DIRECTORIES(SYSTEM ${CLANG_STD_INCLUDE_DIR})
	ENDIF()
	IF(${STDLIB} STREQUAL "libstdc++" OR (${CMAKE_CXX_COMPILER} MATCHES "^.*c[+][+]$" OR ${CMAKE_CXX_COMPILER} MATCHES "^(.*/)g[+][+]$"))
		INCLUDE_DIRECTORIES(SYSTEM ${GCC_C_INCLUDE_DIR})
		FIND_PACKAGE(std_gcc REQUIRED)
		INCLUDE_DIRECTORIES(SYSTEM ${GCC_STD_INCLUDE_DIR})
	ENDIF()
ENDIF()
# generic standard library support, put it as compiler option:
IF(${STDLIB} MATCHES "^.+$")
	SET(STDLIB_FLAG		"-stdlib=${STDLIB}")
ENDIF()

function(stringAppendIfDne appendTo str)
  if("${${appendTo}}" STREQUAL "")
    set(${appendTo} ${str} PARENT_SCOPE)
  else()
    string(FIND ${${appendTo}} ${str} pos)
    if(${pos} EQUAL -1)
      set(${appendTo} "${${appendTo}} ${str}" PARENT_SCOPE)
    endif()
  endif()
endfunction()

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

function(checkCompilerFlags flagVar flags)
  separate_arguments(flags)
  foreach(flag ${flags})
    string(REPLACE "-" "_" flag_ ${flag})
    string(REPLACE "=" "_" flag_ ${flag_})
    if(flagVar MATCHES ".*CXX_FLAGS.*")
      check_cxx_compiler_flag("${flag}" has_cxx${flag_})
      if(has_cxx${flag_})
        stringAppendIfDne(${flagVar} "${flag}")
      endif()
    elseif(flagVar MATCHES ".*C_FLAGS.*")
      check_c_compiler_flag("${flag}" has_c${flag_})
      if(has_c${flag_})
        stringAppendIfDne(${flagVar} "${flag}")
      endif()
    endif()
  endforeach()
  set(${flagVar} "${${flagVar}}" PARENT_SCOPE)
endfunction()

#
# C++ options
#
IF(${CMAKE_CXX_COMPILER} MATCHES ".*clang.*")
	IF (PROFILING)
		MESSAGE(WARNING		"Profiling and clang breaks deterministic builds")
		SET (PROFILE_GET           "-fprofile-instr-generate -g")
		SET (PROFILE_GET_FILE      "RaptorQ.profraw")
		SET (PROFILE_SET_FILE      "RaptorQ.profdata")
		SET (PROFILE_SET           "-fprofile-instr-use=${PROFILE_SET_FILE}")
	ENDIF()
	IF(LTO)
		checkCompilerFlags(CMAKE_CXX_FLAGS "-flto -fuse-ld=gold")
		checkCompilerFlags(CMAKE_C_FLAGS "-flto -fuse-ld=gold")
		FIND_PROGRAM(LLVM_AR "llvm-ar")
		FIND_PROGRAM(LLVM_NM "llvm-nm")
		FIND_PROGRAM(LLVM_RANLIB "llvm-ranlib")
		SET(CMAKE_AR ${LLVM_AR})
		SET(LINK_FLAGS "-rng-seed=${SEED} --build-id=none")
		SET(CMAKE_NM ${LLVM_NM})
		SET(CMAKE_RANLIB ${LLVM_RANLIB})
	ENDIF()
	checkCompilerFlags(CMAKE_CXX_FLAGS "${STDLIB_FLAG} ${DETERMINISTIC} -std=c++11 -fno-rtti -fno-exceptions -Wall -pedantic -Weverything -Wno-c++98-compat-pedantic -Wno-c++98-compat -Wno-padded -fstack-protector-all -fstrict-aliasing")
	checkCompilerFlags(CMAKE_CXX_FLAGS_DEBUG "-O0 -g")
	checkCompilerFlags(CMAKE_CXX_FLAGS_MINSIZEREL "-Os -DNDEBUG")
	checkCompilerFlags(CMAKE_CXX_FLAGS_RELEASE "-Ofast -fvisibility=hidden -fvisibility-inlines-hidden -DNDEBUG -fwrapv -Wformat -Wformat-security -fPIE -fPIC")
	checkCompilerFlags(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-Ofast -g -fvisibility=hidden -fvisibility-inlines-hidden -DNDEBUG -fwrapv -Wformat -Wformat-security -fPIE -fPIC")
	ADD_DEFINITIONS(-DUSING_CLANG)
	SET(USE_CLANG 1)
ELSE()
	# warn: clang+ and g++ get the same *g++ regexp, keep the "else" above or match the "/" in the regexp
	IF(${CMAKE_CXX_COMPILER} MATCHES "^.*c[+][+]$" OR ${CMAKE_CXX_COMPILER} MATCHES "^(.*/)g[+][+]$")
	IF (PROFILING)
		SET (PROFILE_GET           "-fprofile-generate")
		# gcc creates a couple more files than clang for prifiling. Track them.
		SET (PROFILE_GET_FILE
CMakeFiles/RaptorQ_static_profiling.dir/src/Graph.cpp.gcda
CMakeFiles/RaptorQ_static_profiling.dir/src/Parameters.cpp.gcda
CMakeFiles/RaptorQ_static_profiling.dir/src/Rand.cpp.gcda
CMakeFiles/RaptorQ_static_profiling.dir/src/Precode_Matrix.cpp.gcda
CMakeFiles/RaptorQ_static_profiling.dir/src/Precode_Matrix_solver.cpp.gcda
CMakeFiles/RaptorQ_static_profiling.dir/src/cRaptorQ.cpp.gcda
CMakeFiles/test_c_profiled.dir/test/test_c.c.gcda)
		SET (PROFILE_SET_FILE      ${PROFILE_GET_FILE})
		SET (PROFILE_SET           "-fprofile-use")
	ENDIF()
	IF(LTO)
		checkCompilerFlags(CMAKE_CXX_FLAGS "-flto -fuse-ld=gold")
		checkCompilerFlags(CMAKE_C_FLAGS "-flto -fuse-ld=gold")
		FIND_PROGRAM(GCC_AR "gcc-ar")
		FIND_PROGRAM(GCC_NM "gcc-nm")
		FIND_PROGRAM(GCC_RANLIB "gcc-ranlib")
		SET(CMAKE_AR ${GCC_AR})
		SET(LINK_FLAGS "-D")
		SET(CMAKE_NM ${GCC_NM})
		SET(CMAKE_RANLIB ${GCC_RANLIB})
	ENDIF()
	checkCompilerFlags(CMAKE_CXX_FLAGS "-std=c++11 ${DETERMINISTIC} -fno-rtti -fno-exceptions -Wno-unknown-pragmas -Wall -pedantic -Wno-padded -fstack-protector-all -fstrict-aliasing -fwrapv")
	checkCompilerFlags(CMAKE_CXX_FLAGS_DEBUG "-O0 -g -Wno-aggressive-loop-optimizations -ftrapv -fsanitize=undefined -fsanitize=shift -fsanitize=integer-divide-by-zero -fsanitize=vla-bound -fsanitize=null -fsanitize=return -fsanitize=signed-integer-overflow -fisolate-erroneous-paths-dereference -fisolate-erroneous-paths-attribute")
	checkCompilerFlags(CMAKE_CXX_FLAGS_MINSIZEREL "-Os -DNDEBUG")
	checkCompilerFlags(CMAKE_CXX_FLAGS_RELEASE "-Ofast -fvisibility=hidden -fvisibility-inlines-hidden -O4 -DNDEBUG -Wformat -Wformat-security -fPIE -fPIC")
	checkCompilerFlags(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-Ofast -g -fvisibility=hidden -fvisibility-inlines-hidden -O4 -DNDEBUG -Wformat -Wformat-security -fPIE -fPIC")
	ENDIF()
	# else I do not know your compiler options :)
ENDIF()
#
# We hae a C interface example to build.
# set the proper options for that, too
#
IF(${CMAKE_C_COMPILER} MATCHES ".*clang.*")
	checkCompilerFlags(CMAKE_C_FLAGS "${DETERMINISTIC} -std=c11 -Wall -pedantic -Weverything -Wno-padded -fstack-protector-all -fstrict-aliasing -Wno-disabled-macro-expansion")
	checkCompilerFlags(CMAKE_C_FLAGS_DEBUG "-O0 -g")
	checkComiplerFlags(CMAKE_C_FLAGS_MINSIZEREL "-Os -fvisibility=hidden -fvisibility-inlines-hidden -DNDEBUG -fwrapv -Wformat -Wformat-security -fPIE -fPIC")
	checkCompilerFlags(CMAKE_C_FLAGS_RELEASE "-Ofast -fvisibility=hidden -fvisibility-inlines-hidden -DNDEBUG -fwrapv -Wformat -Wformat-security -fPIE -fPIC")
	ADD_DEFINITIONS(-DUSING_CLANG)
ELSE()
	# warn: clang+ and g++ get the same *g++ regexp, keep the "else" above or match the "/" in the regexp
	IF(${CMAKE_C_COMPILER} MATCHES "^.*cc$" OR ${CMAKE_C_COMPILER} MATCHES "^(.*/)gcc$")

	checkCompilerFlags(CMAKE_C_FLAGS "-std=c11 ${DETERMINISTIC} -Wno-unknown-pragmas -Wall -pedantic -Wno-padded -fstack-protector-all -fstrict-aliasing -fwrapv")
	checkCompilerFlags(CMAKE_C_FLAGS_DEBUG "-O0 -g -Wno-aggressive-loop-optimizations -ftrapv -fsanitize=undefined -fsanitize=shift -fsanitize=integer-divide-by-zero -fsanitize=vla-bound -fsanitize=null -fsanitize=return -fsanitize=signed-integer-overflow -fisolate-erroneous-paths-dereference -fisolate-erroneous-paths-attribute")
	checkCompilerFlags(CMAKE_C_FLAGS_MINSIZEREL "-Os -DNDEBUG")
	checkCompilerFlags(CMAKE_C_FLAGS_RELEASE "-Ofast -fvisibility=hidden -DNDEBUG -Wformat -Wformat-security -fPIE -fPIC")
	ENDIF()
	# else I do not know your compiler options :)
ENDIF()

IF(APPLE)
  MESSAGE(STATUS "MacOSX system found. WARN: not tested!")
  ADD_DEFINITIONS(-DUNIX)
ENDIF()
IF(UNIX)
  MESSAGE(STATUS "UNIX system found")
  ADD_DEFINITIONS(-DUNIX)
ELSEIF(WIN32)
  MESSAGE(STATUS "Windows system found. WARN: not tested!")
  ADD_DEFINITIONS(-DWIN32)
ENDIF()


SET(SOURCES	src/Precode_Matrix_solver.cpp
		src/Precode_Matrix.cpp
		src/Rand.cpp
		src/Parameters.cpp
		src/Graph.cpp
		src/cRaptorQ.cpp)

SET(HEADERS	src/Interleaver.hpp
		src/multiplication.hpp
		src/table2.hpp
		src/degree.hpp
		src/common.hpp
		src/Encoder.hpp
		src/Decoder.hpp
		src/Rand.hpp
		src/Precode_Matrix.hpp
		src/Parameters.hpp
		src/Graph.hpp
		src/De_Interleaver.hpp
		src/RaptorQ.hpp
		src/cRaptorQ.h)

SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)

# LATEX documentation
add_subdirectory(${CMAKE_SOURCE_DIR}/doc)

IF(PROFILING)
	# we need to build a profiling version of the static library.
	# then we will run the profiling,  and then finally rebuild
	# everything with the profiling optimizations

	# build the static library for profiling
	ADD_LIBRARY(RaptorQ_static_profiling STATIC ${SOURCES} ${HEADERS})
	# ar: add deterministic build
	SET_TARGET_PROPERTIES (
		RaptorQ_static_profiling
		PROPERTIES
		LINK_FLAGS "-Wl,-z,now,-z,relro -pie"
	)
	TARGET_LINK_LIBRARIES(RaptorQ_static_profiling  stdc++)
	SET_TARGET_PROPERTIES(RaptorQ_static_profiling PROPERTIES COMPILE_FLAGS ${PROFILE_GET})

	# build the C example test for profiling
	ADD_EXECUTABLE(test_c_profiled test/test_c.c)
	ADD_DEPENDENCIES(test_c_profiled RaptorQ_static_profiling)
	SET_TARGET_PROPERTIES(test_c_profiled PROPERTIES COMPILE_FLAGS ${PROFILE_GET})
	TARGET_LINK_LIBRARIES(test_c_profiled RaptorQ_static_profiling)
	STRING(SUBSTRING ${STDLIB} 3 -1 STDLIB_NAME)
	TARGET_LINK_LIBRARIES(test_c_profiled ${STDLIB_NAME})
	TARGET_LINK_LIBRARIES(test_c_profiled m) # math library
	TARGET_LINK_LIBRARIES(test_c_profiled ${CMAKE_THREAD_LIBS_INIT})
	SET_TARGET_PROPERTIES(test_c_profiled PROPERTIES LINK_FLAGS "${PROFILE_GET}")

	# run the profiling and create the profile
	IF(USE_CLANG)
		add_custom_command(
			OUTPUT ${PROFILE_GET_FILE}
			COMMAND LLVM_PROFILE_FILE=${PROFILE_GET_FILE} ./test_c_profiled
			DEPENDS test_c_profiled
			COMMENT "Running profiling test..."
			VERBATIM
		)
	ELSE()
		add_custom_command(
			OUTPUT ${PROFILE_GET_FILE}
			COMMAND ./test_c_profiled
			DEPENDS test_c_profiled
			COMMENT "Running profiling test..."
			VERBATIM
		)
	ENDIF()
	IF(USE_CLANG)
		# the profile must be translated into readable form
		add_custom_command(
			OUTPUT ${PROFILE_SET_FILE}
			COMMAND llvm-profdata merge -output=${PROFILE_SET_FILE} ${PROFILE_GET_FILE}
			DEPENDS ${PROFILE_GET_FILE}
			COMMENT "Creating profile data..."
			VERBATIM
		)
	ENDIF()
	add_custom_target(
		profile ALL
		DEPENDS ${PROFILE_SET_FILE}
	)
	# now we can rebuild the test, the static library with the profiling optimizations.
ENDIF()

ADD_LIBRARY(RaptorQ_static STATIC ${SOURCES} ${HEADERS})
SET_TARGET_PROPERTIES (
	RaptorQ_static
	PROPERTIES
	LINK_FLAGS "-Wl,-z,now,-z,relro -pie"
)
IF(PROFILING)
	ADD_DEPENDENCIES(RaptorQ_static profile)
	SET_TARGET_PROPERTIES(RaptorQ_static PROPERTIES COMPILE_FLAGS ${PROFILE_SET})
ENDIF()
SET(INSTALL_LIBS RaptorQ_static)

ADD_EXECUTABLE(make_deterministic src/deterministic.cpp)
add_custom_command(
  OUTPUT deterministic.run
  COMMAND make_deterministic $<TARGET_FILE:RaptorQ_static>
  DEPENDS RaptorQ_static
  COMMENT "Removing creation date from library..."
  VERBATIM
)
add_custom_target(
  Make_static_deterministic ALL
  DEPENDS deterministic.run
)

IF(BUILD_SHARED_LIB)
  ADD_DEFINITIONS(-DRAPTORQ_DLL)
  ADD_DEFINITIONS(-DRAPTORQ_DLL_EXPORTS)

  ADD_LIBRARY(RaptorQ SHARED ${SOURCES} ${HEADERS})
  # add profiling dependency if requested
  IF(PROFILING)
  	ADD_DEPENDENCIES(RaptorQ profile)
	SET_TARGET_PROPERTIES(RaptorQ PROPERTIES COMPILE_FLAGS ${PROFILE_SET})
  ENDIF()
  SET_TARGET_PROPERTIES (
	RaptorQ
	PROPERTIES
	LINK_FLAGS "-Wl,-z,now,-z,relro"
  )
  LIST(APPEND INSTALL_LIBS RaptorQ)
  SET(theRaptorQLib RaptorQ)
ELSE()
  SET(theRaptorQLib RaptorQ_static)
ENDIF()

# add main tests
ADD_EXECUTABLE(libRaptorQ-test EXCLUDE_FROM_ALL test/rfc_test.cpp)
ADD_DEPENDENCIES(libRaptorQ-test ${theRaptorQLib})
TARGET_LINK_LIBRARIES(libRaptorQ-test ${CMAKE_THREAD_LIBS_INIT})
# link the library we just built, too
TARGET_LINK_LIBRARIES(libRaptorQ-test ${theRaptorQLib})
# see EXCLUDE_FROM_ALL in cmake-properties, Properties on Targets:
# "Installing a target with EXCLUDE_FROM_ALL set to true has undefined behavior."
#INSTALL(TARGETS libRaptorQ-test RUNTIME DESTINATION bin/ OPTIONAL)
ADD_CUSTOM_TARGET(tests DEPENDS libRaptorQ-test)

# build examples
# C interface
ADD_EXECUTABLE(test_c EXCLUDE_FROM_ALL test/test_c.c)
ADD_DEPENDENCIES(test_c ${theRaptorQLib})
TARGET_LINK_LIBRARIES(test_c ${CMAKE_THREAD_LIBS_INIT})
# link the library we just built, too
TARGET_LINK_LIBRARIES(test_c ${theRaptorQLib})
# CPP interface
ADD_EXECUTABLE(test_cpp EXCLUDE_FROM_ALL test/test_cpp.cpp)
ADD_DEPENDENCIES(test_cpp ${theRaptorQLib})
TARGET_LINK_LIBRARIES(test_cpp ${CMAKE_THREAD_LIBS_INIT})
# link the library we just built, too
TARGET_LINK_LIBRARIES(test_cpp ${theRaptorQLib})
ADD_CUSTOM_TARGET(examples DEPENDS test_c test_cpp)

ADD_CUSTOM_TARGET(everything DEPENDS Make_static_deterministic tests examples docs)


INSTALL(FILES ${HEADERS} DESTINATION include/RaptorQ/)
INSTALL(TARGETS ${INSTALL_LIBS} EXPORT libraptorq-targets
	 LIBRARY DESTINATION lib COMPONENT libraries
	 ARCHIVE DESTINATION lib COMPONENT libraries
	 RUNTIME DESTINATION bin COMPONENT libraries)
INSTALL(EXPORT libraptorq-targets DESTINATION lib/cmake)

