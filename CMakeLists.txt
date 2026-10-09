cmake_minimum_required(VERSION 3.7)

project(pt2-clone)

option(EXTERNAL_LIBFLAC "use external(system) flac library" OFF)
option(IRIX_STATIC_SDL2 "link SDL2 statically on IRIX" OFF)

find_package(SDL2 REQUIRED)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${pt2-clone_SOURCE_DIR}/release/other/")

file(GLOB pt2-clone_SRC
    "${pt2-clone_SOURCE_DIR}/src/*.c"
    "${pt2-clone_SOURCE_DIR}/src/modloaders/*.c"
    "${pt2-clone_SOURCE_DIR}/src/smploaders/*.c"
    "${pt2-clone_SOURCE_DIR}/src/gfx/*.c"
)

add_executable(pt2-clone ${pt2-clone_SRC})

target_include_directories(pt2-clone SYSTEM
    PRIVATE ${SDL2_INCLUDE_DIRS})

if("${SDL2_LIBRARIES}" STREQUAL "")
    message(WARNING "SDL2_LIBRARIES wasn't set, manually setting to SDL2::SDL2")
    set(SDL2_LIBRARIES "SDL2::SDL2")
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "IRIX" OR CMAKE_SYSTEM_NAME STREQUAL "IRIX64")
    message(STATUS "IRIX detected!")
    if(IRIX_STATIC_SDL2)
        if(TARGET SDL2::SDL2-static)
            set(SDL2_LIBRARIES SDL2::SDL2-static)
        else()
            # no static target exported, use libSDL2.a + the dependencies sdl2-config reports
            find_program(SDL2_CONFIG_EXECUTABLE sdl2-config)
            if(NOT SDL2_CONFIG_EXECUTABLE)
                message(FATAL_ERROR "IRIX_STATIC_SDL2: sdl2-config not found")
            endif()
            execute_process(COMMAND ${SDL2_CONFIG_EXECUTABLE} --prefix
                OUTPUT_VARIABLE SDL2_PREFIX OUTPUT_STRIP_TRAILING_WHITESPACE)
            execute_process(COMMAND ${SDL2_CONFIG_EXECUTABLE} --static-libs
                OUTPUT_VARIABLE SDL2_STATIC_LIBS OUTPUT_STRIP_TRAILING_WHITESPACE)
            find_library(SDL2_STATIC_LIBRARY NAMES libSDL2.a
                HINTS ${SDL2_PREFIX}/lib32 ${SDL2_PREFIX}/lib /usr/sgug/lib32)
            if(NOT SDL2_STATIC_LIBRARY)
                message(FATAL_ERROR "IRIX_STATIC_SDL2: libSDL2.a not found")
            endif()
            # replace -lSDL2 with the static archive so the shared library isn't picked
            separate_arguments(SDL2_STATIC_LIBS UNIX_COMMAND "${SDL2_STATIC_LIBS}")
            list(REMOVE_ITEM SDL2_STATIC_LIBS -lSDL2 -lSDL2main)
            set(SDL2_LIBRARIES ${SDL2_STATIC_LIBRARY} ${SDL2_STATIC_LIBS})
        endif()
        message(STATUS "Linking SDL2 statically: ${SDL2_LIBRARIES}")
        # libSDL2.a already fills most of the 64KB $gp-relative small-data area,
        # so keep our own globals out of it (avoids "relocation truncated to fit: R_MIPS_GPREL16")
        target_compile_options(pt2-clone PRIVATE -G0)
        target_link_libraries(pt2-clone -G0)
    endif()
    target_link_libraries(pt2-clone  -Wl,-rpath-link=/usr/lib32 -Wl,-rpath=/usr/lib32:/usr/sgug/lib32 -lGLcore -laudio -Wl,--allow-shlib-undefined -pthread ${SDL2_LIBRARIES} )
    target_compile_options(pt2-clone PRIVATE ${SDL2_CFLAGS} -I/usr/sgug/lib/gcc/mips-sgi-irix6.5/9/include -I/usr/include -I/usr/sgug/include )
else()
    target_link_libraries(pt2-clone PRIVATE m ${SDL2_LIBRARIES})
endif()
    
target_compile_definitions(pt2-clone
    PRIVATE HAS_LIBFLAC)
    
if(EXTERNAL_LIBFLAC)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(FLAC REQUIRED IMPORTED_TARGET flac)
    target_compile_definitions(pt2-clone
        PRIVATE EXTERNAL_LIBFLAC)
    target_link_libraries(pt2-clone
        PRIVATE PkgConfig::FLAC)
else()
    file(GLOB flac_SRCS
        "${pt2-clone_SOURCE_DIR}/src/libflac/*.c")
    target_sources(pt2-clone PRIVATE ${flac_SRCS})
endif()

install(TARGETS pt2-clone
    RUNTIME DESTINATION bin)
