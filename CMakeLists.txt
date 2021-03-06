CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
PROJECT(MOOSE)

project(moose)

if(POLICY CMP0048)
    cmake_policy(SET CMP0048 OLD )
endif(POLICY CMP0048)

# If from command line, version info is not passed, use the git to generate a
# version file. If GIT fails, use the previous known version.
find_program(GIT_EXEC "git")
message( STATUS "Looking for git ${GIT_EXEC}" )
if( (NOT MOOSE_VERSION) AND GIT_EXEC)
    execute_process( 
        COMMAND ${GIT_EXEC} describe --tags --long
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE MOOSE_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
endif( )
message( STATUS "Building version ${MOOSE_VERSION}" )

# Options to pass down to moose-core
option(WITH_DOC "Build documentation" OFF)
option(DEBUG "Build with DEBUG support" OFF)
option(WITH_BOOST "Use Boost libraries instead of GSL" OFF)
option(WITH_CUDA "Use CUDA/GPU" OFF)
option(WITH_MPI "Use MPI (experimental)" OFF)
option(WITH_GUI "Install moose-gui. Works only with python2." ON )

# On python3. Do not install GUI. 
find_package(PythonInterp REQUIRED)
if(PYTHON_VERSION_MAJOR VERSION_EQUAL 3)
    message(STATUS "moose-gui does not support python3 yet. Disabling" )
    set(WITH_GUI OFF)
endif( )

# On debian or ubuntu, --install-layout=deb needs to be passed.
if(UNIX AND NOT APPLE)
    find_program(LSB_RELEASE lsb_release)
    if(LSB_RELEASE)
        execute_process(COMMAND ${LSB_RELEASE} -is
            OUTPUT_VARIABLE LSB_RELEASE_ID_SHORT
            OUTPUT_STRIP_TRAILING_WHITESPACE
            )
        if(${LSB_RELEASE_ID_SHORT} STREQUAL "Ubuntu" 
                OR ${LSB_RELEASE_ID_SHORT} STREQUAL "Debian" )
            message(STATUS "Debian based LINUX: ${LSB_RELEASE_ID_SHORT}" )
            set(CMAKE_PYMOOSE_ARGS "-DDISTUTILS_EXTRA_ARGS=\"--install-layout=deb\"" )
        endif( )
    endif( )
endif( )



add_custom_target(moose ALL)


set(PYMOOSE_SOURCE_DIR "${CMAKE_SOURCE_DIR}/moose-core")
set(PYMOOSE_BUILD_DIR ${CMAKE_BINARY_DIR}/__moose-core_build)
set(PYMOOSE_INSTALL_DIR ${CMAKE_BINARY_DIR}/__moose-core_install)

if(WITH_GUI)
    set(MOOSE_GUI_DIR ${CMAKE_SOURCE_DIR}/moose-gui)
    set(MOOSE_GUI_INSTALL_DIR ${CMAKE_BINARY_DIR}/__moose-gui_install)
endif()

set(MOOSE_EXAMPLE_DIR ${CMAKE_SOURCE_DIR}/moose-examples)
set(MOOSE_EXAMPLE_INSTALL_DIR ${CMAKE_BINARY_DIR}/__moose-examples_install)

file(MAKE_DIRECTORY ${PYMOOSE_BUILD_DIR})

# Its a good target since we can not be sure of python module extension on
# different platform.
set(OUTPUT_MOOSEBIN ${PYMOOSE_BUILD_DIR}/moose.bin)

configure_file( ${CMAKE_SOURCE_DIR}/cmake/build_moose-core.sh.in 
    ${CMAKE_BINARY_DIR}/build_moose-core.sh )

# Build pymoose module.
add_custom_target( moose-core ALL DEPENDS ${OUTPUT_MOOSEBIN} )
add_custom_command( OUTPUT ${OUTPUT_MOOSEBIN}
    COMMAND MAKE=$(MAKE) bash -c ${CMAKE_BINARY_DIR}/build_moose-core.sh
    VERBATIM
    )
add_dependencies(moose moose-core)

if(WITH_GUI)
    # Now build moose-gui and moose-examples.
    # Make sure each script is compilable. Else report an error.
    add_custom_target( gui ALL 
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${MOOSE_GUI_DIR} ${MOOSE_GUI_INSTALL_DIR} 
        COMMAND ${PYTHON_EXECUTABLE} -m compileall -q ${MOOSE_GUI_INSTALL_DIR}
        COMMENT "Building moose-gui"
        VERBATIM
        )

    add_custom_target( examples ALL 
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${MOOSE_EXAMPLE_DIR} ${MOOSE_EXAMPLE_INSTALL_DIR}
        COMMAND ${PYTHON_EXECUTABLE} -m compileall -q ${MOOSE_EXAMPLE_INSTALL_DIR}
        VERBATIM
        )
    add_dependencies(moose gui examples)
    add_dependencies(examples  gui)
endif()


## moose-gui
## TODO: moose-gui should be a python module.
set(MOOSE_GUI_DIR ${CMAKE_SOURCE_DIR}/moose-gui)

install(DIRECTORY ${PYMOOSE_INSTALL_DIR}/
    DESTINATION ${CMAKE_INSTALL_PREFIX}
    PATTERN ".git" EXCLUDE
    PATTERN "*.pyc" EXCLUDE
    )

if(WITH_GUI)
    install(DIRECTORY ${MOOSE_GUI_INSTALL_DIR}/
        DESTINATION lib/moose/gui
        PATTERN ".git*" EXCLUDE
        PATTERN "*.pyc" EXCLUDE
        )

    install(DIRECTORY ${MOOSE_EXAMPLE_DIR}/
        DESTINATION lib/moose/moose-examples
        PATTERN ".git*" EXCLUDE 
        PATTERN "_travis*" EXCLUDE
        PATTERN ".travis*" EXCLUDE
        PATTERN "*.pyc" EXCLUDE
        )
endif()

configure_file( 
    ${CMAKE_SOURCE_DIR}/cmake/moosegui.in
    ${CMAKE_BINARY_DIR}/moosegui 
    )
install(PROGRAMS ${CMAKE_BINARY_DIR}/moosegui DESTINATION bin)

# UNINSTALL
add_custom_command(OUTPUT __uninstall_moose-core__
    COMMAND xargs rm -rf < install_manifest.txt
    COMMAND pip uninstall -y moose 
    COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_INSTALL_PREFIX}/bin/moose.bin
    )
add_custom_target(uninstall DEPENDS __uninstall_moose-core__ )

if(WITH_GUI)
    add_custom_command(OUTPUT  __uninstall_moose-gui__
        COMMAND ${CMAKE_COMMAND} -E remove_directory
        ${CMAKE_INSTALL_PREFIX}/lib/moose/gui
        COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_INSTALL_PREFIX}/bin/moose
        )
    #add_dependencies( uninstall __uninstall_moose-gui__ )
endif()

################################################################################
# CTEST 
#################################################################################
enable_testing( )
add_test( NAME test_pymoose_sanity 
    COMMAND ${PYTHON_EXECUTABLE} -c "import moose"
    )

if(WITH_GUI)
    add_test( NAME test_gui_sanity 
        COMMAND ${PYTHON_EXECUTABLE} ${MOOSE_GUI_INSTALL_DIR}/mgui.py 
        WORKING_DIRECTORY ${MOOSE_GUI_INSTALL_DIR}
        )
endif()

set_tests_properties( test_pymoose_sanity 
    PROPERTIES ENVIRONMENT "PYTHONPATH=${PYMOOSE_BUILD_DIR}/python"
    )
