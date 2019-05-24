cmake_minimum_required(VERSION 3.4)

set(CMAKE_MACOSX_RPATH 1)

project(TinyEMU)

set(CMAKE_CXX_STANDARD 11)

set(CMAKE_C_FLAGS "-Wall -Wextra")
set(CMAKE_C_FLAGS_DEBUG  "-g")
set(CMAKE_C_FLAGS_RELEASE "-O3 --llvm-opts 2 -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -MMD -fno-strict-aliasing -DCONFIG_FS_NET")

add_library(js STATIC
        jsemu.c
        softfp.c
        virtio.c
        fs.c
        fs_net.c
        fs_wget.c
        fs_utils.c
        simplefb.c
        pci.c
        json.c
        block_net.c
        iomem.c
        cutils.c
        aes.c
        sha256.c)

if(NOT DEFINED CMAKE_INSTALL_BINDIR)
    set(CMAKE_INSTALL_BINDIR ${CMAKE_BINARY_DIR}/bin)
endif()

file(GLOB emu_SRC riscv_cpu.c riscv_machine.c machine.c elf.c)

function(add_emu_executable ARCH SYSTEM)
    if(SYSTEM STREQUAL "WASM")
        set(target_name riscvemu${ARCH}-wasm)
    else()
        set(target_name riscvemu${ARCH})
    endif()

    add_executable(${target_name} ${emu_SRC})
    target_link_libraries(${target_name} js)
    target_include_directories(${target_name} PRIVATE $ENV{EMSCRIPTEN}/system/include)

    set_target_properties(${target_name} PROPERTIES LINK_FLAGS
        "--memory-init-file 0 \
--closure 0 \
-s MODULARIZE=1 \
-s NO_EXIT_RUNTIME=1 \
-s FORCE_FILESYSTEM=1 \
-s EXPORTED_FUNCTIONS=['_console_queue_char','_vm_start','_fs_import_file','_display_key_event','_display_mouse_event','_display_wheel_event','_net_write_packet','_net_set_carrier'] \
-s EXTRA_EXPORTED_RUNTIME_METHODS=['FS','ccall','cwrap','getValue'] \
-s BINARYEN_TRAP_MODE=clamp \
-s ASSERTIONS=1 \
--js-library ${CMAKE_SOURCE_DIR}/js/lib.js \
-s DISABLE_EXCEPTION_CATCHING=0")

    if(SYSTEM STREQUAL "WASM")
        set_property(TARGET ${target_name} APPEND PROPERTY LINK_FLAGS " -s WASM=1 -s ALLOW_MEMORY_GROWTH=1 -s TOTAL_MEMORY=67108864")
    else()
        set_property(TARGET ${target_name} APPEND PROPERTY LINK_FLAGS " -s WASM=0")
    endif()

    set_property(TARGET ${target_name} APPEND PROPERTY COMPILE_FLAGS "-DMAX_XLEN=${ARCH} -DCONFIG_RISCV_MAX_XLEN=${ARCH}")

    install(TARGETS ${target_name}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_BINDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

    if(SYSTEM STREQUAL "WASM")
        install(FILES ${CMAKE_BINARY_DIR}/${target_name}.wasm DESTINATION ${CMAKE_INSTALL_BINDIR})
    endif()

endfunction(add_emu_executable)

add_emu_executable(32 "WASM")
add_emu_executable(32 "ASM.JS")
add_emu_executable(64 "WASM")
add_emu_executable(64 "ASM.JS")

message(${CMAKE_BINARY_DIR})
install(FILES ${CMAKE_SOURCE_DIR}/js/lib.js DESTINATION ${CMAKE_INSTALL_BINDIR})
