cmake_minimum_required(VERSION 2.8)

set(PROJECT_NAME
        incblocks)


project(${PROJECT_NAME})

set(CMAKE_CXX_FLAGS
        "${CMAKE_CXX_FLAGS} -std=c++14 -fsanitize=address -fno-omit-frame-pointer")

set(CMAKE_C_FLAGS
        "${CMAKE_C_FLAGS} -fsanitize=address -fno-omit-frame-pointer -DLEAK_SANITIZER")

#

if (WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wa,-mbig-obj -O3")
endif()


if (NOT INCOGNITO_DIR)
    set(INCOGNITO_DIR ~/incognito)
endif()

message(STATUS INCOGNITO_DIR ": ${INCOGNITO_DIR}")

set(INCOGNITO_SOURCE_DIR ${INCOGNITO_DIR}
        CACHE PATH "Path to the root directory for Incognito")

# set location of incognito build tree
set(INCOGNITO_BUILD_DIR ${INCOGNITO_SOURCE_DIR}/build/release/
        CACHE PATH "Path to the build directory for Incognito")

set(MY_CMAKE_DIR "${CMAKE_CURRENT_LIST_DIR}/cmake"
        CACHE PATH "The path to the cmake directory of the current project")

list(APPEND CMAKE_MODULE_PATH "${MY_CMAKE_DIR}")

set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${INCOGNITO_BUILD_DIR}"
        CACHE PATH "Add Incognito directory for library searching")

include(MyUtils)

find_package(Incognito)

# find boost
find_package(Boost COMPONENTS
        system
        filesystem
        thread
        date_time
        chrono
        regex
        serialization
        program_options
        date_time
        REQUIRED)



if(APPLE)
    include_directories(/usr/local/opt/openssl/include)
    link_directories(/usr/local/opt/openssl/lib)
endif()


if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin" OR WIN32)
    add_library(unbound STATIC IMPORTED)
    set_property(TARGET unbound PROPERTY IMPORTED_LOCATION ${INCOGNITO_BUILD_DIR}/external/unbound/libunbound.a)
endif()

# include boost headers
include_directories(${Boost_INCLUDE_DIRS})
include_directories("ext/mstch/include")
include_directories("ext/crow")

# add ext/ subfolder
add_subdirectory(ext/)

# add src/ subfolder
add_subdirectory(src/)


set(SOURCE_FILES
        main.cpp)

#ADD_CUSTOM_TARGET(driver DEPENDS src/templates/index.html)

add_executable(${PROJECT_NAME}
        ${SOURCE_FILES})

create_git_version()

configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates ${CMAKE_CURRENT_BINARY_DIR}/templates)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/css ${CMAKE_CURRENT_BINARY_DIR}/templates/css)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/partials ${CMAKE_CURRENT_BINARY_DIR}/templates/partials)

set(LIBRARIES
        myirm
        myext
        mstch
        wallet
        blockchain_db
        cryptonote_core
        cryptonote_protocol
        cryptonote_basic
        daemonizer
        cncrypto
        blocks
        lmdb
        ringct
        common
        mnemonics
        easylogging
        checkpoints
        version
        epee
        ${Boost_LIBRARIES}
        pthread
        unbound
        curl
        crypto
        ssl)

if(APPLE)
    set(LIBRARIES ${LIBRARIES} "-framework IOKit")
else()
    set(LIBRARIES ${LIBRARIES} atomic)
endif()

if (NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin" AND NOT WIN32)
    set(LIBRARIES ${LIBRARIES} unwind)
endif()

if (WIN32)
    set(LIBRARIES ${LIBRARIES}
            wsock32
            ntdll
            ws2_32
            Iphlpapi
            )
else()
    set(LIBRARIES ${LIBRARIES} dl)
endif()

target_link_libraries(${PROJECT_NAME} ${LIBRARIES})
