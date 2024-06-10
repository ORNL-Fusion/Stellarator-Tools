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
