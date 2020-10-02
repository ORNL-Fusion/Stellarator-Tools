cmake_minimum_required (VERSION 3.14)

project (stellarator_tools)

enable_testing ()

include (FetchContent)
find_package (Git)

#  Define a macro to register new projects.
macro (register_project name dir url default_tag)
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
#  registered in the sub project CMakeList.txt. Not sue how this should handel
#  multiple targets in a project yet. Name must match the subproject pull_
#  dependency.
            add_custom_target(
                pull_${name}
                ALL
                COMMAND ${GIT_EXECUTABLE} pull
                WORKING_DIRECTORY ${${name}_SOURCE_DIR}
            )
        endif ()

# Any thing after the the tag is assumed to be a project dependency.
        set (depends "${ARGN}")
        foreach (project IN LISTS depends)
            set (BUILD_${project} ON)
        endforeach ()
    endif ()
endmacro ()

#  Registers the projects. By specifying a name, directory, git url, tag and
#  option dependencies.
#register_project(example_project
#                 PROJECT_DIR
#                 https://url.to.project
#                 project_tag (Default to master)
#                 SUB) #Optional Sub project directories.

# Register MAKEGRID
register_project(makegrid
                 MAKEGRID
                 https://github.com/ORNL-Fusion/MAKEGRID.git
                 master
                 LIBSTELL)

# Register LIBSTELL
register_project(libstell
                 LIBSTELL
                 https://github.com/ORNL-Fusion/LIBSTELL.git
                 master)
