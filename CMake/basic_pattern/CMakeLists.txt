# CMake minimum version
CMAKE_MINIMUM_REQUIRED ( VERSION <version> )

 
# Project name / version
PROJECT ( "<Project Name>" )
SET ( PROJECT_VERSION_MAJOR <Major> )
SET ( PROJECT_VERSION_MINOR <Minor> )
 
# Build Configuration
SET ( CMAKE_BUILD_TYPE <Debug|Release> )
SET ( CMAKE_VERBOSE_MAKEFILE <true|false> )
 
# Binary file name / source files 
SET ( OUTPUT_ELF
        "${CMAKE_PROJECT_NAME}-${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.out"
    )
SET ( SRC_FILES
        <SOURCE_FILE>
        <SOURCE_FILE>
        ...
    )

# Compiler
SET ( CMAKE_C_COMPILER g++ )
                                                 
# Common Header Files(-I)
INCLUDE_DIRECTORIES ( <Dir> <Dir> ... )
                                                 
# Common compile option / linking option
ADD_COMPILE_OPTIONS ( <COMPILE_OPT> <COMPILE_OPT> ... )
SET ( CMAKE_EXE_LINKER_FLAGS "<LINK_OPT> <LINK_OPT> ..." )
                                                 
# Common Link Library(-l)
LINK_LIBRARIES( <Lib> <Lib> ... )
                                                 
# Common Link Library Directory(-L)
LINK_DIRECTORIES ( <Dir> <Dir> ... )
                                                 
# "Debug" options
SET ( CMAKE_C_FLAGS_DEBUG "<Compile_Opt> <Compile_Opt> ..." )
SET ( CMAKE_EXE_LINKER_FLAGS_DEBUG "<Link_Opt> <Link_Opt> ..." )
                                                 
# "Release" options
SET ( CMAKE_C_FLAGS_RELEASE "<Compile_Opt> <Compile_Opt> ..." )
SET ( CMAKE_EXE_LINKER_FLAGS_RELEASE "<Link_Opt> <Link_Opt> ..." )
                                                 
# Output Directory
SET ( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BUILD_TYPE} )
SET ( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BUILD_TYPE}/lib )
SET ( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BUILD_TYPE}/lib )
                                                 
#Build Target Binaray
ADD_EXECUTABLE( ${OUTPUT_ELF} ${SRC_FILES} )
