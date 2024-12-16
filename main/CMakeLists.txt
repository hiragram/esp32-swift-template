###############################################
# ESP-IDFのコンポーネントとしてプロジェクトを登録する
###############################################
idf_component_register(
    # C++のソースファイルがないので空ファイルを指定
    SRCS /dev/null
    # プロジェクト内のカレントディレクトリをprivateなインクルード元として指定
    PRIV_INCLUDE_DIRS "." 
)

###############################################
# ターゲットのアーキテクチャを検証する
###############################################
# ターゲット名(IDF_TARGET)とアーキテクチャ(IDF_BUILD_ARCH)を取得
idf_build_get_property(target IDF_TARGET)
idf_build_get_property(arch IDF_BUILD_ARCH)

# Embedded Swiftではxtensaアーキテクチャをサポートしていない
if("${arch}" STREQUAL "xtensa")
    message(FATAL_ERROR "Not supported target: ${target}")
endif()

###############################################
# コンパイラフラグの抽出
###############################################
# -marchフラグ: コンパイラのターゲット命令セットアーキテクチャを指定するフラグ。例: -march=riscv32
string(REGEX MATCH "-march=[^ ]+" march_flag "${CMAKE_C_FLAGS}")
# ベンダー固有の拡張を削除。例: _x*
string(REGEX REPLACE "_x[^ ]*" "" march_flag "${march_flag}")

# -mabiフラグ: ABIを指定するフラグ。無指定の場合はデフォルトで -mabi=ilp32 を使用。
string(REGEX MATCH "-mabi=[^ ]+" mabi_flag "${CMAKE_C_FLAGS}")
if("${mabi_flag}" STREQUAL "")
    set(mabi_flag "-mabi=ilp32")
endif()

###############################################
# 不要なコンパイルオプションを削除
###############################################
# デフォルトのCOMPILE_OPTIONSにはC/C++用のフラグが含まれているが、Swiftコンパイラはこれらをサポートしないため、削除。
get_target_property(var ${COMPONENT_LIB} COMPILE_OPTIONS)
set_target_properties(${COMPONENT_LIB} PROPERTIES COMPILE_OPTIONS "")

###############################################
# Cヘッダーの検索パスをSwift用に設定
###############################################
# SwiftでC/Objective-Cと連携するために、C/C++のヘッダーファイル検索パスを-Xccフラグで指定。
set(SWIFT_INCLUDES)
foreach(dir ${CMAKE_C_IMPLICIT_INCLUDE_DIRECTORIES})
    string(CONCAT SWIFT_INCLUDES ${SWIFT_INCLUDES} "-Xcc ")
    string(CONCAT SWIFT_INCLUDES ${SWIFT_INCLUDES} "-I${dir} ")
endforeach()

###############################################
# Embedded Swift用コンパイル設定
###############################################
# -target riscv32-none-none-eabi
#   - RISC-V向けのターゲット指定
# -Xfrontend -enable-experimental-feature Embedded
#   - Embedded Swiftの機能を有効化
# -Osize
#   - 最適化レベルを最小化に設定
# -Xccフラグ
#   - C/C++のコンパイラフラグ(-marchや-mabi)をSwiftコンパイラに渡す
# -import-bridging-header
#   - SwiftからC/Objective-Cの関数を呼び出すためのヘッダーファイルを指定
target_compile_options(${COMPONENT_LIB} PUBLIC "$<$<COMPILE_LANGUAGE:Swift>:SHELL:
    -target riscv32-none-none-eabi
    -Xfrontend -function-sections -enable-experimental-feature Embedded -wmo -parse-as-library -Osize
    -Xcc ${march_flag} -Xcc ${mabi_flag} -Xcc -fno-pic -Xcc -fno-pie
    
    -pch-output-dir /tmp
    -Xfrontend -enable-single-module-llvm-emission

    ${SWIFT_INCLUDES}

    -import-bridging-header ${CMAKE_CURRENT_LIST_DIR}/BridgingHeader.h
>")

###############################################
# Swiftのサポートを有効化
###############################################
# Swiftコンパイラが動作することを明示的に指定
set(CMAKE_Swift_COMPILER_WORKS YES)
# wholemodule: モジュール全体をまとめてコンパイルする。(Embedded Swiftで必須)
set(CMAKE_Swift_COMPILATION_MODE_DEFAULT wholemodule)
set(CMAKE_Swift_COMPILATION_MODE wholemodule)
enable_language(Swift)

###############################################
# Swiftファイルの登録
###############################################
target_sources(${COMPONENT_LIB}
    PRIVATE
    Main.swift
)

###############################################
# 不要なセクションを削除
###############################################
# .swift_modhashセクションの削除。組み込み環境では不要なセクションを削除し、ファームウェアのサイズを削減。
add_custom_command(
    TARGET ${COMPONENT_LIB}
    POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} --remove-section .swift_modhash
            $<TARGET_FILE:${COMPONENT_LIB}> $<TARGET_FILE:${COMPONENT_LIB}>
)