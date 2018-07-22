cmake_minimum_required(VERSION 3.4.1)

add_library( v8_base STATIC IMPORTED )
set_target_properties( v8_base PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_base.a )

add_library( v8_snapshot STATIC IMPORTED )
set_target_properties( v8_snapshot PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_snapshot.a )

add_library( native-lib SHARED src/main/cpp/native-lib.cpp )
target_include_directories( native-lib PRIVATE ${CMAKE_SOURCE_DIR}/libs/include )

target_link_libraries( native-lib v8_base v8_snapshot log )