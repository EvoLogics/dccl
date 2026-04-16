set(PROTOS
  option_extensions.proto
 )

# can't use the plugin before we generate it
protobuf_generate_cpp_no_dccl(PROTO_SRCS PROTO_HDRS ${PROTOS})

add_executable(protoc-gen-dccl pb_plugin.cpp ${PROTO_SRCS} ${PROTO_HDRS})
target_include_directories(protoc-gen-dccl PRIVATE ${dccl_INC_DIR})
target_link_libraries(protoc-gen-dccl protobuf::libprotobuf protobuf::libprotoc Threads::Threads)
install(TARGETS protoc-gen-dccl DESTINATION ${CMAKE_INSTALL_BINDIR})
