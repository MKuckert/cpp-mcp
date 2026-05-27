cmake_minimum_required(VERSION 3.10)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

set(TARGET clibridge)
add_executable(${TARGET} main.cpp)
target_link_libraries(${TARGET} PRIVATE mcp)
target_include_directories(${TARGET} PRIVATE ${CMAKE_SOURCE_DIR}/include)
if(OPENSSL_FOUND)
    target_link_libraries(${TARGET} PRIVATE ${OPENSSL_LIBRARIES})
endif()
