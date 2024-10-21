cmake_minimum_required (VERSION 3.14)

project (stellarator_tools)

enable_testing ()

include (FetchContent)
find_package (Git)

#-------------------------------------------------------------------------------
#  Setup build types.
#-------------------------------------------------------------------------------
set_property (CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
              Debug
              Release
              Sanitized
)

#  Make sure the prefix path appears as a ccmake variable.
set (CMAKE_PREFIX_PATH "" CACHE PATH "Search paths for dependency look up.")

#-------------------------------------------------------------------------------
#  Sanitizer options
#-------------------------------------------------------------------------------
add_library (sanitizer INTERFACE)
target_compile_options (sanitizer
                        INTERFACE
                        $<$<CONFIG:Sanitized>:-g>
)

macro (register_sanitizer_option name default)
    string (TOUPPER ${name} upper_name)
    
    option (SANITIZE_${upper_name} "Enable the ${name} sanitizer" ${default})

    target_compile_options (sanitizer
                            INTERFACE
                            $<$<CONFIG:Sanitized>:$<$<BOOL:${SANITIZE_${upper_name}}>:-fsanitize=${name}>>
    )
    target_link_options (sanitizer
                         INTERFACE
                         $<$<CONFIG:Sanitized>:$<$<BOOL:${SANITIZE_${upper_name}}>:-fsanitize=${name}>>
    )
endmacro ()

register_sanitizer_option (address ON)
register_sanitizer_option (leak OFF)
register_sanitizer_option (memory OFF)
register_sanitizer_option (thread OFF)
register_sanitizer_option (undefined ON)
register_sanitizer_option (float-divide-by-zero ON)

#-------------------------------------------------------------------------------
#  Trap options
#-------------------------------------------------------------------------------
add_library (traps INTERFACE)

macro (register_trap_option name)
    string (TOUPPER ${name} upper_name)

    option (TRAP_${upper_name} "Enable the ${name} trap" OFF)

    target_compile_options (traps
                            INTERFACE
                            $<$<AND:$<BOOL:${TRAP_${upper_name}}>,$<Fortran_COMPILER_ID:GNU>,$<COMPILE_LANGUAGE:Fortran>>:-ffpe-trap=${name}>
    )
endmacro ()

register_trap_option (invalid)
register_trap_option (zero)
register_trap_option (overflow)
register_trap_option (inexact)
register_trap_option (denormal)

#-------------------------------------------------------------------------------
#  Trap options
#-------------------------------------------------------------------------------
add_library (checks INTERFACE)

macro (register_checks_option name)
    string (TOUPPER ${name} upper_name)

    option (CHECK_${upper_name} "Enable the ${name} checks" OFF)

    target_compile_options (checks
                            INTERFACE
                            $<$<AND:$<BOOL:${CHECK_${upper_name}}>,$<Fortran_COMPILER_ID:GNU>,$<COMPILE_LANGUAGE:Fortran>>:-fcheck=${name}>
    )
endmacro ()

register_checks_option (recursion)
register_checks_option (pointer)
register_checks_option (mem)
register_checks_option (do)
register_checks_option (bounds)

#-------------------------------------------------------------------------------
#  Setup access method.
#-------------------------------------------------------------------------------
option (USE_SSH "Use ssh to access git repos" OFF)
if (${USE_SSH})
    set (URL_PROTO "git@")
    set (URL_SEP ":")
else ()
    set (URL_PROTO "https://")
    set (URL_SEP "/")
endif ()

#-------------------------------------------------------------------------------
#  Define a macro to register new projects.
#-------------------------------------------------------------------------------
macro (register_project name url default_tag)
    string (TOUPPER ${name} dir)

    option (BUILD_${dir} "Build ${name} subproject." OFF)
    set (BUILD_TAG_${dir} ${default_tag} CACHE STRING "Name of the tag to checkout.")

    if (BUILD_${dir})
#  Set up the sub project repository.
        FetchContent_Declare(
            ${name}
            GIT_REPOSITORY ${url}
            GIT_TAG origin/${BUILD_TAG_${dir}}
            SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${dir}
        )

        FetchContent_MakeAvailable(${name})

        if (GIT_FOUND)
#  By default cmake clones projects in a headless state. After the repo is
#  cloned checkout the requested tag so repo is in a working state.
            execute_process(
                COMMAND ${GIT_EXECUTABLE} checkout ${BUILD_TAG_${dir}}
                WORKING_DIRECTORY ${${name}_SOURCE_DIR}
            )

#  Add a taraget to pull the latest version before building. Note dependency is
#  registered in the sub project CMakeList.txt. Not sure how this should handle
#  multiple targets in a project yet. Name must match the subproject pull_
#  dependency.
            add_custom_target(
                pull_${name}
                ALL
                COMMAND ${GIT_EXECUTABLE} pull
                WORKING_DIRECTORY ${${name}_SOURCE_DIR}
            )
        endif ()

# Anything after the the tag is assumed to be a project dependency.
        set (depends "${ARGN}")
        foreach (project IN LISTS depends)
            string (TOUPPER ${project} dep)
            set (BUILD_${dep} ON CACHE BOOL "Build ${dep}" FORCE)
        endforeach ()
    endif ()
endmacro ()

#-------------------------------------------------------------------------------
#  Build Documentation
#-------------------------------------------------------------------------------
find_package(Doxygen)

if (DOXYGEN_FOUND)
    set (DOXYGEN_PROJECT_NAME "Stellarator-Tools")
    set (DOXYGEN_PROJECT_LOGO ${CMAKE_CURRENT_SOURCE_DIR}/images/flux_surfaces.png)
    set (DOXYGEN_EXCLUDE ${CMAKE_CURRENT_SOURCE_DIR}/build ${CMAKE_CURRENT_SOURCE_DIR}/images)
    set (DOXYGEN_ALIASES item{3}=\"<tr><td width=20%>\\fixed_width{\\1}</td><td width=50%>\\2</td><td width=30%>\\ref \\3</td></tr>\",item2{2}=\"<tr><td width=30%>\\1</td><td width=70%>\\2</td></tr>\",item3{3}=\"<tr><td width=20%>\\fixed_width{\\1}</td><td width=5%>\\2</td><td width=75%>\\3</td></tr>\",table_section{2}=\"\\subsection \\1 \\2^^<table border=1px width=80%>\",begin_table=\"<table border=1px width=80%>\",end_table=\"</table>\",table_subsection{2}=\"</table>\\subsubsection \\1 \\2^^<table border=1px width=80%>\",header{3}=\"<table border=1px width=80%><tr><td width=20%>\\fixed_width{\\1}</td><td width=50%>\\2</td><td width=30%>\\3</td></tr></table>\",header2{3}=\"<table border=1px width=80%><tr><td width=20%>\\fixed_width{\\1}</td><td width=5%>\\2</td><td width=75%>\\3</td></tr></table>\",fixed_width{1}=\"<tt>\\1</tt>\",header3{3}=\"<table border=1px width=80%><tr><td width=20%>\\fixed_width{\\1}</td><td width=20%>\\2</td><td width=60%>\\3</td></tr></table>\",item4{3}=\"<tr><td width=20%>\\fixed_width{\\1}</td><td width=20%>\\2</td><td width=60%>\\3</td></tr>\")
    set (DOXYGEN_IMAGE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/images)
    set (DOXYGEN_GENERATE_TREEVIEW YES)

    doxygen_add_docs (doc)
endif ()

#-------------------------------------------------------------------------------
#  Registers the projects. By specifying a name, directory, git url, tag and
#  optional dependencies.
#
#  register_project(example_project
#                   PROJECT_DIR
#                   https://url.to.project
#                   project_tag (Default to master)
#                   SUB) #Optional Sub project directories.
#-------------------------------------------------------------------------------

# Register BOOZ_XFORM
register_project(booz_xform
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/BOOZ_XFORM.git
                 master
                 libstell)

# Register V3FIT
register_project(v3fit
                 ${URL_PROTO}github.com${URL_SEP}cianciosa/V3FIT.git
                 master
                 siesta
                 v3rfun
                 lgrid)

# Register V3RFUN
register_project(v3rfun
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/V3RFUN.git
                 master
                 libstell)

# Register SIESTA
register_project(siesta
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/SIESTA.git
                 master
                 parvmec
                 bmw)

# Register LGRID
register_project(lgrid
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/LGRID.git
                 master
                 libstell)

# Register SURFACE
register_project(surface
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/SURFACE.git
                 master
                 bmw)

# Register BMW
register_project(bmw
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/BMW.git
                 master
                 libstell)

# Register DESCUR
register_project(descur
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/DESCUR.git
                 master
                 libstell)

# Register PARVMEC
register_project(parvmec
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/PARVMEC.git
                 master
                 makegrid
                 libstell)

# Register MAKEGRID
register_project(makegrid
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/MAKEGRID.git
                 master
                 libstell)

# Register LIBSTELL
register_project(libstell
                 ${URL_PROTO}github.com${URL_SEP}ORNL-Fusion/LIBSTELL.git
                 master)
