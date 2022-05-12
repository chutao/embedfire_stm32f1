set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_SYSTEM_VERSION 1)
set(CMAKE_CROSSCOMPILING ON)
cmake_minimum_required(VERSION 3.22)

# specify cross-compilers and tools
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER  arm-none-eabi-gcc)
set(CMAKE_AR arm-none-eabi-ar)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_OBJDUMP arm-none-eabi-objdump)
set(SIZE arm-none-eabi-size)
set(GDB arm-none-eabi-gdb)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# project settings
project(embedfire_f103 C CXX ASM)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

#Uncomment for hardware floating point
#add_compile_definitions(ARM_MATH_CM4;ARM_MATH_MATRIX_CHECK;ARM_MATH_ROUNDING)
#add_compile_options(-mfloat-abi=hard -mfpu=fpv4-sp-d16)
#add_link_options(-mfloat-abi=hard -mfpu=fpv4-sp-d16)

#Uncomment for software floating point
#add_compile_options(-mfloat-abi=soft)

add_compile_options(-mcpu=cortex-m3 -mthumb)
add_compile_options(-ffunction-sections -fdata-sections -fno-common -fmessage-length=0)
#add_compile_options(-specs=nano.specs -specs=nosys.specs -Wl,-u,_printf_float,-u,_scanf_float)

# uncomment to mitigate c++17 absolute addresses warnings
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-register")

# Enable assembler files preprocessing
add_compile_options($<$<COMPILE_LANGUAGE:ASM>:-x$<SEMICOLON>assembler-with-cpp>)

if ("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
    message(STATUS "Maximum optimization for speed")
    add_compile_options(-Ofast)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
    message(STATUS "Maximum optimization for speed, debug info included")
    add_compile_options(-Ofast -g)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "MinSizeRel")
    message(STATUS "Maximum optimization for size")
    add_compile_options(-Os)
else ()
    message(STATUS "Minimal optimization, debug info included")
    add_compile_options(-Og -g)
endif ()

include_directories(
    Core/Inc
    Drivers/STM32F1xx_HAL_Driver/Inc
    Drivers/STM32F1xx_HAL_Driver/Inc/Legacy
    Drivers/CMSIS/Device/ST/STM32F1xx/Include
    Drivers/CMSIS/Include
    Middlewares/Third_Party/FatFs/src
    Middlewares/Third_Party/FatFs/src/drivers
    Middlewares/ST/STM32_USB_Device_Library/Core/Inc
    Middlewares/ST/STM32_USB_Device_Library/Class/MSC/Inc
    FATFS/Target
    FATFS/App
    USB_DEVICE/App
    USB_DEVICE/Target
)

add_definitions(-DUSE_HAL_DRIVER -DSTM32F103xE)

file(GLOB_RECURSE SOURCES
  "Startup/*.s"
  "Core/*.c"
  "Drivers/*.c"
  "Middlewares/*.c"
  "FATFS/*.c"
  "USB_DEVICE/*.c"
)

set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/STM32F103ZETx_FLASH.ld)

add_link_options(-Wl,-gc-sections,--print-memory-usage,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map,--cref)
add_link_options(-mcpu=cortex-m3 -mthumb -lnosys -lc)
add_link_options(-specs=nano.specs -specs=nosys.specs -Wl,-u,_printf_float,-u,_scanf_float)
add_link_options(-T ${LINKER_SCRIPT})

add_executable(${PROJECT_NAME}.elf ${SOURCES} ${LINKER_SCRIPT})

set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
        COMMENT "Building ${HEX_FILE}
Building ${BIN_FILE}")

add_custom_target(flash
        COMMAND openocd -f interface/jlink.cfg -c "transport select swd" -f target/stm32f1x.cfg -c "init"
          -c "reset halt" -c "sleep 100" -c "wait_halt 2" -c "flash write_image erase ${PROJECT_NAME}.bin 0x08000000"
          -c "sleep 100" -c "verify_image ${PROJECT_NAME}.bin 0x08000000" -c "sleep 100" -c "reset run" -c shutdown
        COMMENT "Flash ${HEX_FILE}"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

set(INVOKE_TERM xterm -e)
if(WIN32)
    set(INVOKE_TERM start)
endif(WIN32)

add_custom_target(debug
        COMMAND xterm -e ${GDB}
          -iex 'target extended | openocd -f ${CMAKE_CURRENT_SOURCE_DIR}/openocd.cfg -c \"gdb_port pipe\"'
          -iex 'mon halt'
          -iex 'mon reset'
          ${PROJECT_NAME}.elf
        COMMENT "Debug ${PROJECT_NAME}.elf"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

