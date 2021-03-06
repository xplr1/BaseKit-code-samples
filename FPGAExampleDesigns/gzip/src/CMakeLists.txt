set(DEVICE_SOURCE_FILE gzipkernel.cpp)
set(DEVICE_HEADER_FILE gzipkernel.hpp) 
set(HOST_SOURCE_FILE gzip.cpp crc32.cpp WriteGzip.cpp CompareGzip.cpp)

set(TARGET_NAME gzip)
set(EMULATOR_TARGET ${TARGET_NAME}.fpga_emu)
set(FPGA_TARGET ${TARGET_NAME}.fpga)
set(REPORTS_TARGET ${TARGET_NAME}_report)

# Intel supported FPGA Boards and their names
set(A10_PAC_BOARD_NAME "intel_a10gx_pac:pac_a10")
set(S10_PAC_BOARD_NAME "intel_s10sx_pac:pac_s10")

# set FPGA_BOARD value for the example design
IF (NOT DEFINED FPGA_BOARD)
    MESSAGE(STATUS "\tFPGA_BOARD was not specified, defaulting to using the ${A10_PAC_BOARD_NAME} (A10 PAC) board. Please refer to the README for more information on how to run the design on the S10 PAC board.")
    SET(FPGA_BOARD ${A10_PAC_BOARD_NAME})

ELSEIF(FPGA_BOARD STREQUAL ${A10_PAC_BOARD_NAME})
    MESSAGE(STATUS "\tConfiguring the design to run on the ${A10_PAC_BOARD_NAME} (A10 PAC) board.")
    SET(FPGA_BOARD ${A10_PAC_BOARD_NAME})

ELSEIF(FPGA_BOARD STREQUAL ${S10_PAC_BOARD_NAME})
    MESSAGE(STATUS "\tConfiguring the design to run on the ${S10_PAC_BOARD_NAME} (S10 PAC) board.")
    SET(FPGA_BOARD ${S10_PAC_BOARD_NAME})

ELSE()
    MESSAGE(STATUS "\tAn invalid board name was specified using the FPGA_BOARD flag. The default ${A10_PAC_BOARD_NAME} (A10 PAC) board will be used. Please refer to the README for the list of valid board names.")
    SET(FPGA_BOARD ${A10_PAC_BOARD_NAME})
ENDIF()

#specify -MMD -fsycl-link-targets=... instead of -fintelfpga to workaround known issue; lower report quality 
set(HARDWARE_COMPILE_FLAGS -MMD -fsycl-link-targets=spir64_fpga-unknown-unknown-sycldevice -c)

# use cmake -D USER_HARDWARE_FLAGS=<flags> to set extra flags for FPGA backend compilation
separate_arguments(USER_HARDWARE_FLAGS)
set(HARDWARE_LINK_FLAGS -fintelfpga -Xshardware -Xsno-accessor-aliasing -Xsparallel=2 -Xsseed=1 -Xsboard=${FPGA_BOARD} ${USER_HARDWARE_FLAGS})
set(FINAL_LINK_FLAGS -fintelfpga)

set(EMULATOR_COMPILE_FLAGS "-g0 -fintelfpga -DFPGA_EMULATOR")
set(EMULATOR_LINK_FLAGS -fintelfpga)

# fpga emulator
if(WIN32)
    set(WIN_EMULATOR_TARGET ${EMULATOR_TARGET}.exe)
    add_custom_target(fpga_emu DEPENDS ${WIN_EMULATOR_TARGET})
    separate_arguments(WIN_EMULATOR_COMPILE_FLAGS WINDOWS_COMMAND "${EMULATOR_COMPILE_FLAGS}")
    add_custom_command(OUTPUT ${WIN_EMULATOR_TARGET} 
                   COMMAND ${CMAKE_CXX_COMPILER} ${WIN_EMULATOR_COMPILE_FLAGS} /GX ${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_SOURCE_FILE} ${CMAKE_CURRENT_SOURCE_DIR}/gzip.cpp ${CMAKE_CURRENT_SOURCE_DIR}/crc32.cpp ${CMAKE_CURRENT_SOURCE_DIR}/WriteGzip.cpp ${CMAKE_CURRENT_SOURCE_DIR}/CompareGzip.cpp  -o ${CMAKE_BINARY_DIR}/${WIN_EMULATOR_TARGET}
                   DEPENDS ${DEVICE_SOURCE_FILE} ${HOST_SOURCE_FILE})
else()
    add_executable(${EMULATOR_TARGET} ${DEVICE_SOURCE_FILE} ${HOST_SOURCE_FILE})
    add_custom_target(fpga_emu DEPENDS ${EMULATOR_TARGET})
    set_target_properties(${EMULATOR_TARGET} PROPERTIES COMPILE_FLAGS ${EMULATOR_COMPILE_FLAGS})
    set_target_properties(${EMULATOR_TARGET} PROPERTIES LINK_FLAGS ${EMULATOR_LINK_FLAGS})
endif()

# fpga
if(WIN32)
    add_custom_target(fpga
                  COMMAND echo "FPGA hardware flow is not supported in Windows")
else()
    add_custom_target(fpga DEPENDS ${FPGA_TARGET})
    set(DEVICE_FPGA_OBJ "gzipkernel_fpga.o")
    set(DEVICE_IMAGE_FPGA_OBJ "gzipkernel_fpga.a")
    set(HOST_SOURCE_FILES_WITH_PATH ${CMAKE_CURRENT_SOURCE_DIR}/gzip.cpp ${CMAKE_CURRENT_SOURCE_DIR}/crc32.cpp ${CMAKE_CURRENT_SOURCE_DIR}/WriteGzip.cpp ${CMAKE_CURRENT_SOURCE_DIR}/CompareGzip.cpp)

    add_custom_command(OUTPUT ${DEVICE_FPGA_OBJ}
                       COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS} ${HARDWARE_COMPILE_FLAGS} ${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_SOURCE_FILE} -o ${DEVICE_FPGA_OBJ}
                       DEPENDS ${DEVICE_SOURCE_FILE} ${DEVICE_HEADER_FILE})

    set(OBJ_FILES)
    foreach(HOST_FILE ${HOST_SOURCE_FILES_WITH_PATH})
      set(HOST_FPGA_OBJ ${HOST_FILE}.o)
      add_custom_command(OUTPUT ${HOST_FPGA_OBJ}
                         COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS} ${HARDWARE_COMPILE_FLAGS} ${HOST_FILE} -o ${HOST_FPGA_OBJ}
                         DEPENDS ${HOST_FILE})
      list(APPEND OBJ_FILES ${HOST_FPGA_OBJ})
    endforeach()

    add_custom_command(OUTPUT ${DEVICE_IMAGE_FPGA_OBJ}
                       COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS} ${HARDWARE_LINK_FLAGS} -fsycl-link=image ${DEVICE_FPGA_OBJ} -o ${DEVICE_IMAGE_FPGA_OBJ}
                       DEPENDS ${DEVICE_FPGA_OBJ} ${OBJ_FILES})

    add_custom_command(OUTPUT ${FPGA_TARGET}
                       COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS} ${FINAL_LINK_FLAGS} ${OBJ_FILES} ${DEVICE_IMAGE_FPGA_OBJ} -o  ${CMAKE_BINARY_DIR}/${FPGA_TARGET}
                       DEPENDS ${DEVICE_IMAGE_FPGA_OBJ} ${OBJ_FILES})
endif()

# fpga report
if(WIN32)
    add_custom_target(report DEPENDS ${REPORTS_TARGET} )

    separate_arguments(WIN_FLAGS WINDOWS_COMMAND)
    add_custom_command(OUTPUT ${REPORTS_TARGET}
        COMMAND ${CMAKE_CXX_COMPILER} /EHsc ${WIN_FLAGS} ${HARDWARE_LINK_FLAGS} -fsycl-link ${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_SOURCE_FILE} -o ${CMAKE_BINARY_DIR}/${REPORTS_TARGET}
                      DEPENDS ${DEVICE_SOURCE_FILE} ${DEVICE_HEADER_FILE})

else()
    add_custom_target(report DEPENDS ${REPORTS_TARGET} )

    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_SOURCE_FILE} ${DEVICE_SOURCE_FILE} COPYONLY)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/kernels.hpp kernels.hpp COPYONLY)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${DEVICE_HEADER_FILE} ${DEVICE_HEADER_FILE} COPYONLY)

    add_custom_command(OUTPUT ${REPORTS_TARGET}
                      COMMAND ${CMAKE_CXX_COMPILER} ${CMAKE_CXX_FLAGS} ${HARDWARE_LINK_FLAGS} -fsycl-link ${DEVICE_SOURCE_FILE} -o ${CMAKE_BINARY_DIR}/${REPORTS_TARGET}
                      DEPENDS ${DEVICE_SOURCE_FILE} ${DEVICE_HEADER_FILE} kernels.hpp)
endif()

# run
add_custom_target(run
                  COMMAND ../${TARGET_NAME}.fpga_emu Makefile -o=test.gz
                  DEPENDS ${TARGET_NAME}.fpga_emu)

