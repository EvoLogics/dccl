protobuf_generate_cpp(PROTO_B_SRCS PROTO_B_HDRS test_b.proto)
add_library(test_dyn_protobuf SHARED ${PROTO_B_SRCS} ${PROTO_B_HDRS})
target_compile_definitions(test_dyn_protobuf PRIVATE SHARED_LIBRARY_SUFFIX="${CMAKE_SHARED_LIBRARY_SUFFIX}")
target_link_libraries(test_dyn_protobuf dl dccl protobuf::libprotobuf)

protobuf_generate_cpp(PROTO_A_SRCS PROTO_A_HDRS test_a.proto)
add_executable(dccl_test_dynamic_protobuf dynamic_protobuf.cpp ${PROTO_A_SRCS} ${PROTO_A_HDRS})
target_compile_definitions(dccl_test_dynamic_protobuf PRIVATE SHARED_LIBRARY_SUFFIX="${CMAKE_SHARED_LIBRARY_SUFFIX}")
target_link_libraries(dccl_test_dynamic_protobuf dccl Boost::boost protobuf::libprotobuf ${CMAKE_DL_LIBS})
add_test(NAME dccl_test_dynamic_protobuf
  COMMAND ${dccl_BIN_DIR}/dccl_test_dynamic_protobuf $<TARGET_FILE:test_dyn_protobuf>)
