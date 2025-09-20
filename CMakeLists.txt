cmake_minimum_required(VERSION 4.0)

project(tt)

option(TT_USE_CCACHE "use ccache from the host system")

include(ExternalProject)

set(out "${CMAKE_SOURCE_DIR}/install")
set(dl "${CMAKE_SOURCE_DIR}/download")

ExternalProject_Add(libxml2
    URL "https://download.gnome.org/sources/libxml2/2.15/libxml2-2.15.0.tar.xz"
    URL_HASH SHA256=5abc766497c5b1d6d99231f662e30c99402a90d03b06c67b62d6c1179dedd561
    DOWNLOAD_DIR "${dl}"

    CMAKE_CACHE_ARGS
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/libxml2
)

ExternalProject_Add(zlib
    URL "https://zlib.net/zlib-1.3.1.tar.xz"
    URL_HASH SHA256=38ef96b8dfe510d42707d9c781877914792541133e1870841463bfa73f883e32
    DOWNLOAD_DIR "${dl}"

    CMAKE_GENERATOR Ninja
    CMAKE_CACHE_ARGS
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/zlib
)

ExternalProject_Add(zstd
    URL "https://github.com/facebook/zstd/archive/refs/tags/v1.5.7.tar.gz"
    URL_HASH SHA256=37d7284556b20954e56e1ca85b80226768902e2edabd3b649e9e72c0c9012ee3
    DOWNLOAD_DIR "${dl}"

    CMAKE_GENERATOR Ninja
    SOURCE_SUBDIR build/cmake
    CMAKE_CACHE_ARGS
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/zstd
)

ExternalProject_Add(llvm-mid
    URL "https://github.com/llvm/llvm-project/archive/refs/tags/llvmorg-21.1.1.tar.gz"
    URL_HASH SHA256=5f048351ee63050d7fa45b6a1160768fb222a8d306a89e1344515ef7a4bcd278
    DOWNLOAD_DIR "${dl}"

    SOURCE_SUBDIR llvm
    CMAKE_GENERATOR Ninja
    CMAKE_CACHE_ARGS
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/llvm-mid
        -DLLVM_CCACHE_BUILD:BOOL=${TT_USE_CCACHE}
        -DLLVM_DISTRIBUTION_COMPONENTS:STRING=clang;clang-resource-headers;builtins;runtimes;lld
        -DLLVM_ENABLE_PROJECTS:STRING=clang;lld
        -DLLVM_ENABLE_RUNTIMES:STRING=compiler-rt;libcxx;libcxxabi;libunwind
        -DLLVM_TARGETS_TO_BUILD:STRING=X86

    BUILD_COMMAND ninja install-distribution
    INSTALL_COMMAND ""
)

ExternalProject_Add(llvm-final
    URL "https://github.com/llvm/llvm-project/archive/refs/tags/llvmorg-21.1.1.tar.gz"
    URL_HASH SHA256=5f048351ee63050d7fa45b6a1160768fb222a8d306a89e1344515ef7a4bcd278
    DOWNLOAD_DIR "${dl}"

    SOURCE_SUBDIR llvm
    CMAKE_GENERATOR Ninja
    CMAKE_CACHE_ARGS
        -DCMAKE_BUILD_TYPE:STRING=Release
        -DCMAKE_CXX_COMPILER:STRING=${out}/llvm-mid/bin/clang++
        -DCMAKE_C_COMPILER:STRING=${out}/llvm-mid/bin/clang
        -DCMAKE_EXE_LINKER_FLAGS:STRING=-lc++
        -DCMAKE_INSTALL_PREFIX:STRING=${out}/llvm-final
        -DLLVM_CCACHE_BUILD:BOOL=${TT_USE_CCACHE}
        -DLLVM_DISTRIBUTION_COMPONENTS:STRING=clang;clang-resource-headers;builtins;runtimes;lld
        -DLLVM_ENABLE_LIBCXX:BOOL=ON
        -DLLVM_ENABLE_LLD:BOOL=ON
        -DLLVM_ENABLE_PROJECTS:STRING=clang;lld
        -DLLVM_ENABLE_RUNTIMES:STRING=compiler-rt;libcxx;libcxxabi;libunwind
        -DLLVM_STATIC_LINK_CXX_STDLIB:BOOL=ON
        -DLLVM_TARGETS_TO_BUILD:STRING=X86

    BUILD_COMMAND
        "${CMAKE_COMMAND}" -E env
            LD_LIBRARY_PATH=${out}/llvm-mid/lib/x86_64-unknown-linux-gnu
        ninja install-distribution
    INSTALL_COMMAND ""
)
ExternalProject_Add_StepDependencies(llvm-final build llvm-mid libxml2 zlib zstd)
