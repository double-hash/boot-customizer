cmake_minimum_required(VERSION 2.8)

if(NOT DEFINED CMAKE_TOOLCHAIN_FILE)
    if(DEFINED ENV{VITASDK})
        set(CMAKE_TOOLCHAIN_FILE "$ENV{VITASDK}/share/vita.toolchain.cmake" CACHE PATH "toolchain file")
    else()
        message(FATAL_ERROR "Please define VITASDK to point to your SDK path!")
    endif()
endif()

project(boot-customizer)
include("${VITASDK}/share/vita.cmake" REQUIRED)
set(VITA_APP_NAME ${PROJECT_NAME})
set(VITA_TITLEID  "LVHX00001")
set(VITA_VERSION  "00.01")
set(BUILD_DIR "build")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wl,-q -Wall -fno-lto")

FUNCTION(ADD_RESOURCES out_var)
    SET(result)
    FOREACH(in_f ${ARGN})
        SET(out_f "${CMAKE_CURRENT_BINARY_DIR}/${in_f}.o")
        GET_FILENAME_COMPONENT(out_dir ${out_f} DIRECTORY)
        ADD_CUSTOM_COMMAND(OUTPUT ${out_f}
                COMMAND ${CMAKE_COMMAND} -E make_directory ${out_dir}
                COMMAND ${CMAKE_LINKER} -r -b binary -o ${out_f} ${in_f}
                DEPENDS ${in_f}
                WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
                COMMENT "Building resource ${out_f}"
                VERBATIM
                )
        LIST(APPEND result ${out_f})
    ENDFOREACH()
    SET(${out_var} "${result}" PARENT_SCOPE)
ENDFUNCTION()

file(GLOB res_files RELATIVE
        ${PROJECT_SOURCE_DIR}
        image.png
        )
add_resources(boot_customizer_res ${res_files})

add_executable(${PROJECT_NAME}
        ${boot_customizer_res}
        src/main.c
        )

target_link_libraries(${PROJECT_NAME}
        vita2d
        SceDisplay_stub
        SceGxm_stub
        SceSysmodule_stub
        SceCtrl_stub
        ScePgf_stub
        ScePvf_stub
        SceCommonDialog_stub
        SceAppMgr_stub
        freetype
        png
        jpeg
        z
        m
        c
        )

vita_create_self(eboot.bin ${PROJECT_NAME})
vita_create_vpk(${PROJECT_NAME}.vpk ${VITA_TITLEID} eboot.bin
        VERSION ${VITA_VERSION}
        NAME ${VITA_APP_NAME}
        )

add_custom_target(send
        COMMAND curl -T eboot.bin ftp://$ENV{PSVITAIP}:1337/ux0:/app/${VITA_TITLEID}/
        DEPENDS eboot.bin
        )

add_custom_target(copy
        COMMAND cp eboot.bin F:/app/$ENV{PSVITAIP}/eboot.bin
        DEPENDS eboot.bin
        )

add_custom_target(vpksend
        COMMAND curl -T .vpk ftp://$ENV{PSVITAIP}:1337/ux0:/data/
        DEPENDS ${PROJECT_NAME}.vpk
        )
