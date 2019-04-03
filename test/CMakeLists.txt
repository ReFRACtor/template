set(TEMPLATE_TEST_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
include_directories(${TEMPLATE_TEST_DIR} ${TEMPLATE_LIB_DIR})

# Add each test file to TEST_SOURCES
set(TEST_SOURCES
    #file1_test.cc
)

# Create test executable and link to main library target
add_executable(template_test ${TEST_SOURCES})
target_link_libraries(template_test template ${THIRDPARTY_LIBRARIES})
set_target_properties(template_test PROPERTIES INSTALL_RPATH "${PROJECT_BINARY_DIR}/lib;${CMAKE_INSTALL_PREFIX}/lib")
if(DEFINED ENV{CONDA_PREFIX})
  set_property(TARGET template_test APPEND PROPERTY INSTALL_RPATH "$ENV{CONDA_PREFIX}/lib")
endif(DEFINED ENV{CONDA_PREFIX})
set_target_properties(template_test PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)

# Add test to ctest
add_test(NAME template COMMAND template_test --log_level=test_suite)

# Expose test directory location to top level scope
set(TEMPLATE_TEST_DIR ${TEMPLATE_TEST_DIR} PARENT_SCOPE)
