cmake_minimum_required(VERSION 3.13.0)

project(sdl3-ps3-example VERSION 0.1.0)

file(GLOB all_SRCS
    "${PROJECT_SOURCE_DIR}/src/*.cpp"
)

if(NOT DEFINED ENV{PS3DEV})
    message(FATAL_ERROR "Need environment variable PS3DEV set")
endif()

set(PS3DEV "$ENV{PS3DEV}")

include (${PS3DEV}/share/ps3dev.cmake)

add_executable(${PROJECT_NAME} ${all_SRCS})
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin/ps3)

set(
    PROJECT_LINK_DIRS
    ${PS3DEV}/ppu/lib/
    ${PS3DEV}/portlibs/ppu/lib
    ${PS3DEV}/spu/lib/
)

set(
    PROJECT_LIBRARIES
    SDL3
    SDL3_mixer
    SDL3_ttf
    z
    gcm_sys
    rsx
    sysutil
    rt
    lv2
    io
    audio
    m
)

set(
    PROJECT_INCLUDE_DIRS
    "${PS3DEV}/ppu/include"
    "${PS3DEV}/portlibs/ppu/include"
    "${PS3DEV}/portlibs/ppu/include/SDL3"
    "${PS3DEV}/portlibs/ppu/include/SDL3_mixer"
    "${PS3DEV}/portlibs/ppu/include/SDL3_ttf"
)

target_link_directories(${PROJECT_NAME} PUBLIC ${PROJECT_LINK_DIRS} )
include_directories(${PROJECT_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} PUBLIC ${PROJECT_LIBRARIES} )

target_compile_options(${PROJECT_NAME} PRIVATE
    -mpowerpc64
    -mhard-float
    -fno-pic           # important — PIC breaks OPD on PSL1GHT
    -fno-stack-protector
    -mminimal-toc      #  prevents TOC overflow on PS3
    -fno-exceptions    # PSL1GHT has limited exception support
)

target_link_options(${PROJECT_NAME} PRIVATE
    -Wl,-q             # preserve relocs for OPD
    -Wl,--gc-sections  # ✅ remove unused sections
)

function(add_ps3_build_steps target)
    set(ELF_FILE $<TARGET_FILE:${target}>)
    set(STRIPPED_ELF ${ELF_FILE}.stripped.elf)
    set(SELF_FILE ${ELF_FILE}.self)
    set(FSELF_FILE ${ELF_FILE}.fake.self)

    add_custom_command(
        TARGET ${target}
        POST_BUILD
        COMMAND echo "Running PS3 post-build steps..."

        # 1. Fix OPD relocations FIRST, before anything else
        COMMAND ${PS3DEV}/bin/sprxlinker ${ELF_FILE}

        # 2. Strip AFTER sprxlinker, preserve OPD with -R flag
        COMMAND ${PS3DEV}/ppu/bin/powerpc64-ps3-elf-strip
            --strip-debug          # only strip debug, keep OPD symbols
            ${ELF_FILE}
            -o ${STRIPPED_ELF}

        # 3. Package the STRIPPED elf into SELF
        COMMAND ${PS3DEV}/bin/make_self ${STRIPPED_ELF} ${SELF_FILE}
        COMMAND ${PS3DEV}/bin/fself ${STRIPPED_ELF} ${FSELF_FILE}

        COMMAND echo "Done: ${SELF_FILE}"
    )
endfunction()

add_ps3_build_steps(${PROJECT_NAME})
