
set(SRC     
    thread_pool.cpp
    )

if (NOT MACORO_HAS_STD_STOP_TOKEN)
message("adding stop_token impl, ${MACORO_HAS_STD_STOP_TOKEN}")
    LIST(APPEND SRC 
        detail/stop_source.cpp
        detail/stop_state.cpp
        detail/stop_token.cpp
    )
endif()
if(MSVC)
    LIST(APPEND SRC 
    detail/win32.cpp)
endif()

add_library(macoro STATIC ${SRC} "trace.h" "barrier.h" )



target_include_directories( macoro PUBLIC 
                    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/..>
                    $<INSTALL_INTERFACE:include>) 
target_include_directories( macoro PUBLIC 
                    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/..>
                    $<INSTALL_INTERFACE:>) 
                
configure_file(config.h.in config.h)
                    

if(MACORO_OPTIONAL_LITE_V)
    target_link_Libraries(macoro PUBLIC nonstd::optional-lite)
endif()

if(MACORO_VARIANT_LITE_V)
    target_link_Libraries(macoro PUBLIC nonstd::variant-lite)
endif()

if(APPLE OR UNIX)
    target_compile_options( macoro PRIVATE
        $<$<COMPILE_LANGUAGE:CXX>:-std=c++${MACORO_CPP_VER}>
    )
    target_compile_options(macoro PUBLIC   $<$<COMPILE_LANGUAGE:CXX>:-Wno-error=switch>)
    if(MACORO_PIC)
        target_compile_options(macoro PUBLIC
            "-fPIC"
        )
    endif()
elseif(MSVC)

    target_compile_options( macoro PRIVATE
        $<$<COMPILE_LANGUAGE:CXX>:/std:c++${MACORO_CPP_VER}>
    )
    target_compile_options( macoro PUBLIC
        "/D_ENABLE_EXTENDED_ALIGNED_STORAGE"
    )


    target_link_libraries(macoro PUBLIC ws2_32.lib)
else()
    set_target_properties( macoro 
        PROPERTIES
            CXX_STANDARD ${MACORO_CPP_VER}
            CXX_STANDARD_REQUIRED YES
            CXX_EXTENSIONS NO
    )
endif()
#if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND MACORO_CPP_20)
#    target_compile_options( macoro PUBLIC
#        "-fcoroutines"
#        )
#endif()

if(MACORO_ASAN)
    message("Warning: ASAN is enabled for macoro.")
    if(MSVC)
        target_compile_options(macoro PUBLIC 
            "/fsanitize=address")
    else()
        target_compile_options(macoro PRIVATE 
            "-fsanitize=address")
        target_link_options(macoro PUBLIC "-fsanitize=address")
    endif()
endif()

find_package (Threads)
target_link_libraries(macoro PUBLIC Threads::Threads)



#############################################
#            Install                        #
#############################################

# make cache variables for install destinations
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

configure_file("${CMAKE_CURRENT_LIST_DIR}/../cmake/macoroConfigVersion.cmake.in" "macoroConfigVersion.cmake" COPYONLY)
configure_file("${CMAKE_CURRENT_LIST_DIR}/../cmake/macoroFindDeps.cmake" "macoroFindDeps.cmake" COPYONLY)
configure_file("${CMAKE_CURRENT_LIST_DIR}/../cmake/macoroPreamble.cmake" "macoroPreamble.cmake" COPYONLY)
configure_file("${CMAKE_CURRENT_LIST_DIR}/../cmake/macoroBuildOptions.cmake.in" "macoroBuildOptions.cmake")

# generate the config file that is includes the exports
configure_package_config_file(
  "${CMAKE_CURRENT_LIST_DIR}/../cmake/Config.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/macoroConfig.cmake"
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/macoro
  NO_SET_AND_CHECK_MACRO
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

if(NOT DEFINED macoro_VERSION_MAJOR)
    message("\n\n\n\n warning, macoro_VERSION_MAJOR not defined ${macoro_VERSION_MAJOR}")
endif()

set_property(TARGET macoro PROPERTY VERSION ${macoro_VERSION})

# install the configuration file
install(FILES
          "${CMAKE_CURRENT_BINARY_DIR}/macoroBuildOptions.cmake"
          "${CMAKE_CURRENT_BINARY_DIR}/macoroConfig.cmake"
          "${CMAKE_CURRENT_BINARY_DIR}/macoroConfigVersion.cmake"
          "${CMAKE_CURRENT_BINARY_DIR}/macoroFindDeps.cmake"
          "${CMAKE_CURRENT_BINARY_DIR}/macoroPreamble.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/macoro
)

# install library
install(
    TARGETS macoro 
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    EXPORT macoroTargets)

# install headers
install(DIRECTORY . DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/macoro FILES_MATCHING PATTERN "*.h")


#install config header
install( FILES
    "${CMAKE_CURRENT_BINARY_DIR}/config.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/macoro")

# install config
install(EXPORT macoroTargets
  FILE macoroTargets.cmake
  NAMESPACE macoro::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/macoro
)
 export(EXPORT macoroTargets
       FILE "${CMAKE_CURRENT_BINARY_DIR}/macoroTargets.cmake"
       NAMESPACE macoro::
)

