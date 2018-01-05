# ---[ Google Test
hunter_add_package(GTest)
find_package(GTest CONFIG REQUIRED)

enable_testing()
find_package(Threads REQUIRED)
file(GLOB_RECURSE UNIT_TEST_SOURCE "*.cc")
include_directories(cpp/include)
add_executable(${PROJECT_NAME}_unit_tests ${UNIT_TEST_SOURCE})
set_property(TARGET ${PROJECT_NAME}_unit_tests
  PROPERTY RUNTIME_OUTPUT_DIRECTORY ${PRIVATE_RUNTIME_DIR})
target_link_libraries(${PROJECT_NAME}_unit_tests
  GTest::main
  dmlc
  Threads::Threads
  )
add_test(AllTestsIn${PROJECT_NAME}UnitTests ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${PROJECT_NAME}_unit_tests)
