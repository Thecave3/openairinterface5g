# SPDX-License-Identifier: LicenseRef-CSSL-1.0

find_package(PkgConfig REQUIRED)
# libe3's C API is not stable across minor releases: 0.2.0 adds a correlation
# id to the control and report callbacks, which this agent does not pass.
pkg_check_modules(CLIBE3 REQUIRED libe3>=0.0.10 libe3<0.2.0)

if(NOT CLIBE3_FOUND)
    message(FATAL_ERROR "libe3 not found and required for E3 Agent")
endif()

# Two constraints make pkg_check_modules report the version per module.
message(STATUS "Using for E3AP libe3 version: ${CLIBE3_libe3_VERSION}")

include_directories(${CLIBE3_INCLUDE_DIRS})

add_subdirectory(service_models)
add_library(e3ap e3_agent.c config/e3_config.c)

# target_link_libraries(e3ap PUBLIC OCP_ITTI) # TODO

# pkg-config tells us where libe3 lives; without this the -llibe3 above only
# resolves when libe3 happens to sit in a default linker path.
target_link_directories(e3ap PUBLIC ${CLIBE3_LIBRARY_DIRS})
target_link_libraries(e3ap PUBLIC ${CLIBE3_LIBRARIES} ${SERVICE_MODELS_LIBRARY})

# Export libe3 as a usage requirement of e3ap rather than relying on the
# directory-scoped include_directories() above, which no other directory sees.
# e3_agent.h includes <libe3/c_api.h>, so anything that includes it needs the
# include path.
target_include_directories(e3ap PUBLIC ${CLIBE3_INCLUDE_DIRS})

# PR #204 merged the dApp-report bridge in e3_agent.c behind #ifdef E2_AGENT,
# but nothing defined E2_AGENT for this target, so that code has never been
# compiled. Define it, and put the RAN function's extern header on the include
# path, when the E2 agent is built too.
if(E2_AGENT)
    target_link_libraries(e3ap PUBLIC e2_ran_func_cuup e2_ran_func_du_cucp_cuup)
    target_compile_definitions(e3ap PRIVATE E2_AGENT)
    target_include_directories(e3ap PUBLIC ${OPENAIR_DIR}/openair2/E2AP/RAN_FUNCTION/O-RAN)
endif()
