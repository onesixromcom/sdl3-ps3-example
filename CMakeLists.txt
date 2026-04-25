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

# target_link_directories(${PROJECT_NAME} PUBLIC ${PROJECT_LINK_DIRS} )
target_include_directories(${PROJECT_NAME} PUBLIC
    "${PS3DEV}/portlibs/ppu/include/SDL3"
    "${PS3DEV}/portlibs/ppu/include/SDL3_mixer"
    "${PS3DEV}/portlibs/ppu/include/SDL3_ttf")
target_link_libraries(${PROJECT_NAME} PUBLIC
    SDL3
    SDL3_mixer
    SDL3_ttf
)

ps3_target_setup(${PROJECT_NAME})
ps3_build_self(${PROJECT_NAME})