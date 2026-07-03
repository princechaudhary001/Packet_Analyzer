cmake_minimum_required(VERSION 3.16)
project(PacketAnalyzer VERSION 1.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

include_directories(${CMAKE_SOURCE_DIR}/include)

# Shared sources used by both engine variants
set(COMMON_SOURCES
    src/pcap_reader.cpp
    src/packet_parser.cpp
    src/sni_extractor.cpp
    src/types.cpp
)

# Simple single-threaded engine (dpi_simple)
add_executable(dpi_simple src/main_working.cpp ${COMMON_SOURCES})

# Multi-threaded engine (dpi_engine)
add_executable(dpi_engine src/dpi_mt.cpp ${COMMON_SOURCES})
find_package(Threads REQUIRED)
target_link_libraries(dpi_engine PRIVATE Threads::Threads)

# Original minimal target retained for compatibility
add_executable(packet_analyzer src/main.cpp src/pcap_reader.cpp src/packet_parser.cpp)

if(APPLE)
    # Add any macOS-specific settings here
endif()
