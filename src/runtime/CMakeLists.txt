# Keep these lists in alphabetical order.
set(RUNTIME_CPP
    aarch64_cpu_features
    alignment_128
    alignment_32
    alignment_64
    allocation_cache
    android_clock
    android_host_cpu_count
    android_io
    arm_cpu_features
    cache
    can_use_target
    cuda
    d3d12compute
    destructors
    device_interface
    errors
    fake_get_symbol
    fake_thread_pool
    float16_t
    fuchsia_clock
    fuchsia_host_cpu_count
    fuchsia_yield
    gpu_device_selection
    halide_buffer_t
    hexagon_cache_allocator
    hexagon_cpu_features
    hexagon_dma
    hexagon_dma_pool
    hexagon_host
    ios_io
    linux_clock
    linux_host_cpu_count
    linux_yield
    matlab
    metadata
    metal
    metal_objc_arm
    metal_objc_x86
    mips_cpu_features
    module_aot_ref_count
    module_jit_ref_count
    msan
    msan_stubs
    opencl
    opengl
    opengl_egl_context
    opengl_glx_context
    openglcompute
    osx_clock
    osx_get_symbol
    osx_host_cpu_count
    osx_opengl_context
    osx_yield
    posix_abort
    posix_allocator
    posix_clock
    posix_error_handler
    posix_get_symbol
    posix_io
    posix_print
    posix_threads
    posix_threads_tsan
    powerpc_cpu_features
    prefetch
    profiler
    profiler_inlined
    pseudostack
    qurt_allocator
    qurt_hvx
    qurt_hvx_vtcm
    qurt_init_fini
    qurt_threads
    qurt_threads_tsan
    qurt_yield
    riscv_cpu_features
    runtime_api
    ssp
    to_string
    trace_helper
    tracing
    wasm_cpu_features
    windows_abort
    windows_clock
    windows_cuda
    windows_get_symbol
    windows_io
    windows_opencl
    windows_profiler
    windows_threads
    windows_threads_tsan
    windows_yield
    write_debug_image
    x86_cpu_features
    )

set(RUNTIME_LL
    aarch64
    arm
    arm_no_neon
    d3d12_abi_patch_64
    hvx_128
    hvx_64
    mips
    posix_math
    powerpc
    ptx_dev
    wasm_math
    win32_math
    x86
    x86_avx
    x86_avx2
    x86_sse41
    )

set(RUNTIME_BC
    compute_20
    compute_30
    compute_35
    )

set(RUNTIME_HEADER_FILES
    HalideBuffer.h
    HalidePyTorchCudaHelpers.h
    HalidePyTorchHelpers.h
    HalideRuntime.h
    HalideRuntimeCuda.h
    HalideRuntimeD3D12Compute.h
    HalideRuntimeHexagonDma.h
    HalideRuntimeHexagonHost.h
    HalideRuntimeMetal.h
    HalideRuntimeOpenCL.h
    HalideRuntimeOpenGL.h
    HalideRuntimeOpenGLCompute.h
    HalideRuntimeQurt.h
    )

# Need to create an object library for this because CMake
# doesn't support using target_sources on a target declared
# in a different directory ONLY IF that source was created
# by add_custom_command, as is the case in this directory.
add_library(Halide_initmod OBJECT)

set(CXX_WARNING_FLAGS -Wall -Werror -Wno-unused-function -Wcast-qual)
set(RUNTIME_CXX_FLAGS -O3
    -fno-vectorize -ffreestanding -fno-blocks -fno-exceptions -fno-unwind-tables -fpic
    # Note: we don't want static locals to get thread synchronization stuff.
    -fno-threadsafe-statics)

if (NOT MSVC)
    # -g produces dwarf debugging info, which is not useful on windows
    #  (and fails to compile due to llvm bug 15393)
    list(APPEND RUNTIME_CXX_FLAGS "-g")
endif ()

foreach (i IN LISTS RUNTIME_CPP)
    foreach (j IN ITEMS 32 64)
        if (${j} EQUAL 32)
            if (${i} MATCHES "windows_.*")
                # win32 uses the stdcall calling convention, which is x86-specific
                set(TARGET "i386-unknown-unknown-unknown")
            else ()
                # (The 'nacl' is a red herring. This is just a generic 32-bit little-endian target.)
                set(TARGET "le32-unknown-nacl-unknown")
            endif ()
        else ()
            # generic 64-bit code
            set(TARGET "le64-unknown-unknown-unknown")
        endif ()

        set(RUNTIME_DEFINES -DDEBUG_RUNTIME -DCOMPILING_HALIDE_RUNTIME -DLLVM_VERSION=${LLVM_VERSION} -DBITS_${j})

        set(SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/${i}.cpp")

        set(LL_D "initmod.${i}_${j}_debug.ll")
        set(LL "initmod.${i}_${j}.ll")

        set(BC_D "initmod.${i}_${j}_debug.bc")
        set(BC "initmod.${i}_${j}.bc")

        set(INITMOD_D "_initmod_${i}_${j}_debug.cpp")
        set(INITMOD "_initmod_${i}_${j}.cpp")

        add_custom_command(OUTPUT "${LL_D}"
                           DEPENDS "${SOURCE}"
                           VERBATIM
                           COMMAND clang ${RUNTIME_CXX_FLAGS} ${RUNTIME_DEFINES} -m${j} -target ${TARGET} -emit-llvm -S "$<SHELL_PATH:${SOURCE}>" -o ${LL_D}
                           # Make sure that the output of this command also depends
                           # on the header files that ${SOURCE} uses
                           IMPLICIT_DEPENDS CXX "${SOURCE}" # Note: Only works for makefile generator
                           )
        add_custom_command(OUTPUT "${BC_D}"
                           DEPENDS "${LL_D}"
                           COMMAND llvm-as "${LL_D}" -o "${BC_D}"
                           )
        add_custom_command(OUTPUT "${INITMOD_D}"
                           DEPENDS "${BC_D}" binary2cpp
                           COMMAND binary2cpp "halide_internal_initmod_${i}_${j}_debug" < "${BC_D}" > "${INITMOD_D}"
                           )
        target_sources(Halide_initmod PRIVATE ${INITMOD_D})

        add_custom_command(OUTPUT "${LL}"
                           DEPENDS "${SOURCE}"
                           VERBATIM
                           COMMAND clang ${RUNTIME_CXX_FLAGS} -m${j} -target ${TARGET} -DCOMPILING_HALIDE_RUNTIME -DLLVM_VERSION=${LLVM_VERSION} -DBITS_${j} -emit-llvm -S "$<SHELL_PATH:${SOURCE}>" -o ${LL}
                           )
        add_custom_command(OUTPUT "${BC}"
                           DEPENDS "${LL}"
                           COMMAND llvm-as "${LL}" -o "${BC}"
                           )
        add_custom_command(OUTPUT "${INITMOD}"
                           DEPENDS "${BC}" binary2cpp
                           COMMAND binary2cpp "halide_internal_initmod_${i}_${j}" < "${BC}" > "${INITMOD}"
                           )
        target_sources(Halide_initmod PRIVATE ${INITMOD})
    endforeach ()
endforeach ()


foreach (i IN LISTS RUNTIME_LL)
    set(LL "${CMAKE_CURRENT_SOURCE_DIR}/${i}.ll")

    set(BC "initmod.${i}.bc")

    set(INITMOD "_initmod_${i}.cpp")

    add_custom_command(OUTPUT "${BC}"
                       DEPENDS "${LL}"
                       VERBATIM
                       COMMAND llvm-as "$<SHELL_PATH:${LL}>" -o "${BC}"
                       )
    add_custom_command(OUTPUT "${INITMOD}"
                       DEPENDS "${BC}" binary2cpp
                       COMMAND binary2cpp "halide_internal_initmod_${i}_ll" < "${BC}" > "${INITMOD}"
                       )
    target_sources(Halide_initmod PRIVATE ${INITMOD})
endforeach ()

foreach (i IN LISTS RUNTIME_BC)
    set(INITMOD "_initmod_ptx_${i}.cpp")
    set(RT_BC "${CMAKE_CURRENT_SOURCE_DIR}/nvidia_libdevice_bitcode/libdevice.${i}.10.bc")

    add_custom_command(OUTPUT "${INITMOD}"
                       DEPENDS binary2cpp "${RT_BC}"
                       COMMAND binary2cpp "halide_internal_initmod_ptx_${i}_ll" < "$<SHELL_PATH:${RT_BC}>" > "${INITMOD}"
                       VERBATIM)
    target_sources(Halide_initmod PRIVATE ${INITMOD})
endforeach ()

add_custom_command(OUTPUT "_initmod_inlined_c.cpp"
                   DEPENDS "halide_buffer_t.cpp" binary2cpp
                   COMMAND binary2cpp "halide_internal_initmod_inlined_c" < "${CMAKE_CURRENT_SOURCE_DIR}/halide_buffer_t.cpp" > "_initmod_inlined_c.cpp"
                   )
target_sources(Halide_initmod PRIVATE "_initmod_inlined_c.cpp")

foreach (i IN LISTS RUNTIME_HEADER_FILES)
    string(REPLACE "." "_" SYM_NAME "${i}")
    add_custom_command(OUTPUT "_initmod_${SYM_NAME}.cpp"
                       DEPENDS "${i}" binary2cpp
                       COMMAND binary2cpp "halide_internal_runtime_header_${SYM_NAME}" < "${CMAKE_CURRENT_SOURCE_DIR}/${i}" > "_initmod_${SYM_NAME}.cpp"
                       )
    target_sources(Halide_initmod PRIVATE "_initmod_${SYM_NAME}.cpp")

    configure_file(${i} "${Halide_BINARY_DIR}/include/${i}" COPYONLY)
endforeach ()

##
# Target for the runtime
##

add_library(Halide_Runtime INTERFACE)
add_library(Halide::Runtime ALIAS Halide_Runtime)
target_include_directories(Halide_Runtime INTERFACE $<BUILD_INTERFACE:${Halide_BINARY_DIR}/include>)
set_target_properties(Halide_Runtime PROPERTIES EXPORT_NAME Runtime)

