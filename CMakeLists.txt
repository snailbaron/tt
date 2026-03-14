cmake_minimum_required(VERSION 3.24)

project(tt LANGUAGES NONE)

option(TT_USE_CCACHE "use ccache from the host system")

include(ExternalProject)

set(llvm_url "https://github.com/llvm/llvm-project/archive/refs/tags/llvmorg-21.1.1.tar.gz")
set(llvm_hash SHA256=5f048351ee63050d7fa45b6a1160768fb222a8d306a89e1344515ef7a4bcd278)

set(libxml2_url "https://download.gnome.org/sources/libxml2/2.15/libxml2-2.15.0.tar.xz")
set(libxml2_hash SHA256=5abc766497c5b1d6d99231f662e30c99402a90d03b06c67b62d6c1179dedd561)

# GitHub URLs are more reliable, as old releases don't get deleted. The
# original zlib.net URL is left mostly for traceability.
set(zlib_urls
    "https://github.com/madler/zlib/releases/download/v1.3.2/zlib-1.3.2.tar.xz"
    "https://zlib.net/zlib-1.3.2.tar.xz"
)
set(zlib_hash SHA256=d7a0654783a4da529d1bb793b7ad9c3318020af77667bcae35f95d0e42a792f3)

set(zstd_url "https://github.com/facebook/zstd/archive/refs/tags/v1.5.7.tar.gz")
set(zstd_hash SHA256=37d7284556b20954e56e1ca85b80226768902e2edabd3b649e9e72c0c9012ee3)

set(out "${CMAKE_SOURCE_DIR}/install")
set(dl "${CMAKE_SOURCE_DIR}/download")

set(llvm_options_components_to_build
    -DLLVM_DISTRIBUTION_COMPONENTS:STRING=clang;clang-resource-headers;builtins;runtimes;lld
    -DLLVM_ENABLE_PROJECTS:STRING=clang;lld
    -DLLVM_ENABLE_RUNTIMES:STRING=compiler-rt;libcxx;libcxxabi;libunwind
    -DLLVM_TARGETS_TO_BUILD:STRING=X86
)

set(llvm_options_default_tools
    -DCLANG_DEFAULT_CXX_STDLIB:STRING=libc++
    -DCLANG_DEFAULT_LINKER:STRING=lld
    -DCLANG_DEFAULT_RTLIB:STRING=compiler-rt
    -DCLANG_DEFAULT_UNWINDLIB:STRING=libunwind
)

set(llvm_options_runtime_linkage
    -DCOMPILER_RT_BUILD_BUILTINS:BOOL=ON
    -DLIBCXXABI_ENABLE_SHARED:BOOL=OFF
    -DLIBCXXABI_ENABLE_STATIC_UNWINDER:BOOL=ON
    -DLIBCXXABI_STATICALLY_LINK_UNWINDER_IN_STATIC_LIBRARY:BOOL=ON
    -DLIBCXX_ENABLE_ABI_LINKER_SCRIPT:BOOL=OFF
    -DLIBCXX_ENABLE_SHARED:BOOL=OFF
    -DLIBCXX_ENABLE_STATIC_ABI_LIBRARY:BOOL=ON
    -DLIBUNWIND_ENABLE_SHARED:BOOL=OFF
)

ExternalProject_Add(llvm-mid
    URL ${llvm_url}
    URL_HASH ${llvm_hash}
    DOWNLOAD_DIR "${dl}"

    SOURCE_SUBDIR llvm
    CMAKE_GENERATOR Ninja
    CMAKE_CACHE_ARGS
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/llvm-mid
        -DLLVM_CCACHE_BUILD:BOOL=${TT_USE_CCACHE}

        ${llvm_options_components_to_build}
        ${llvm_options_default_tools}
        ${llvm_options_runtime_linkage}

        -DLLVM_ENABLE_LIBXML2:STRING=OFF
        -DLLVM_ENABLE_ZLIB:STRING=OFF
        -DLLVM_ENABLE_ZSTD:STRING=OFF

    LOG_CONFIGURE ON
    LOG_BUILD ON

    BUILD_COMMAND ninja install-distribution
    INSTALL_COMMAND ""
)

ExternalProject_Add(libxml2
    URL ${libxml2_url}
    URL_HASH ${libxml2_hash}
    DOWNLOAD_DIR "${dl}"

    CMAKE_GENERATOR Ninja
    CMAKE_CACHE_ARGS
        -DBUILD_SHARED_LIBS:BOOL=OFF
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_CXX_COMPILER:STRING=${out}/llvm-mid/bin/clang++
        -DCMAKE_C_COMPILER:STRING=${out}/llvm-mid/bin/clang
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/libxml2

        -DLIBXML2_WITH_ICONV:BOOL=OFF
        -DLIBXML2_WITH_PROGRAMS:BOOL=OFF
        -DLIBXML2_WITH_TESTS:BOOL=OFF
)
ExternalProject_Add_StepDependencies(libxml2 configure llvm-mid)

ExternalProject_Add(zlib
    URL ${zlib_urls}
    URL_HASH ${zlib_hash}
    DOWNLOAD_DIR "${dl}"

    CMAKE_GENERATOR Ninja
    CMAKE_CACHE_ARGS
        -DBUILD_SHARED_LIBS:BOOL=OFF
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_CXX_COMPILER:STRING=${out}/llvm-mid/bin/clang++
        -DCMAKE_C_COMPILER:STRING=${out}/llvm-mid/bin/clang
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/zlib

        -DZLIB_BUILD_TESTING:BOOL=OFF
        -DZLIB_BUILD_SHARED:BOOL=OFF
)
ExternalProject_Add_StepDependencies(zlib configure llvm-mid)

ExternalProject_Add(zstd
    URL ${zstd_url}
    URL_HASH ${zstd_hash}
    DOWNLOAD_DIR "${dl}"

    CMAKE_GENERATOR Ninja
    SOURCE_SUBDIR build/cmake
    CMAKE_CACHE_ARGS
        -DBUILD_SHARED_LIBS:BOOL=OFF
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_CXX_COMPILER:STRING=${out}/llvm-mid/bin/clang++
        -DCMAKE_C_COMPILER:STRING=${out}/llvm-mid/bin/clang
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/zstd
        -DZSTD_BUILD_SHARED:BOOL=OFF
        -DZSTD_BUILD_STATIC:BOOL=ON

        -DZSTD_LEGACY_SUPPORT:BOOL=OFF
        -DZSTD_BUILD_PROGRAMS:BOOL=OFF
        -DZSTD_BUILD_TESTS:BOOL=OFF
)
ExternalProject_Add_StepDependencies(zstd configure llvm-mid)

ExternalProject_Add(llvm-final
    URL ${llvm_url}
    URL_HASH ${llvm_hash}
    DOWNLOAD_DIR "${dl}"

    SOURCE_SUBDIR llvm
    CMAKE_GENERATOR Ninja
    CMAKE_CACHE_ARGS
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/llvm-final
        -DCMAKE_SKIP_RPATH:BOOL=ON
        -DLLVM_CCACHE_BUILD:BOOL=${TT_USE_CCACHE}

        -DCMAKE_CXX_COMPILER:STRING=${out}/llvm-mid/bin/clang++
        -DCMAKE_C_COMPILER:STRING=${out}/llvm-mid/bin/clang

        ${llvm_options_components_to_build}
        ${llvm_options_default_tools}
        ${llvm_options_runtime_linkage}

        -DLIBCXXABI_USE_COMPILER_RT:BOOL=ON
        -DLIBCXXABI_USE_LLVM_UNWINDER:BOOL=ON
        -DLIBCXX_USE_COMPILER_RT:BOOL=ON
        -DLIBUNWIND_USE_COMPILER_RT:BOOL=ON
        -DLLVM_ENABLE_LIBCXX:BOOL=ON
        -DLLVM_ENABLE_LLD:BOOL=ON
        -DLLVM_ENABLE_LTO:BOOL=ON
        -DLLVM_STATIC_LINK_CXX_STDLIB:BOOL=ON

        -DLLVM_USE_STATIC_ZSTD:BOOL=ON
        -DZLIB_USE_STATIC_LIBS:BOOL=ON

        -DLLVM_ENABLE_LIBXML2:STRING=FORCE_ON
        -DLIBXML2_ROOT:PATH=${out}/libxml2

        -DLLVM_ENABLE_ZLIB:STRING=FORCE_ON
        -DZLIB_ROOT:PATH=${out}/zlib

        -DLLVM_ENABLE_ZSTD:STRING=FORCE_ON
        -DZSTD_ROOT:PATH=${out}/zstd

    LOG_CONFIGURE ON
    LOG_BUILD ON

    BUILD_COMMAND
        "${CMAKE_COMMAND}" -E env
            LD_LIBRARY_PATH=${out}/llvm-mid/lib/x86_64-unknown-linux-gnu
        ninja install-distribution
    INSTALL_COMMAND ""
)
ExternalProject_Add_StepDependencies(llvm-final configure llvm-mid)
ExternalProject_Add_StepDependencies(llvm-final build libxml2 zlib zstd)
