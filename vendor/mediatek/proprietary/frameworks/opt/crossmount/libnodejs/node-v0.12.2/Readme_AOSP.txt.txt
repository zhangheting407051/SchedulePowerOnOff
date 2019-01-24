/***Build How To***/

1. Enter alps/ 
2. Clean build (if not yet) 
3. mosesq mmm external/libnodejs/node-v0.12.2/ 
4. mosesq mmm external/openssl/ 

The above 2 steps generate both node and openssl static libraries. 

5. After that put indivisually the 32-bit and 64-bit static libraries into 'external/webserver/jni/armeabi/node32/' and 'external/webserver/jni/armeabi/node64', as below.

cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libnodev8_intermediates/libnodev8_64.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libnode_intermediates/libnode.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libssl_static_intermediates/libssl_static.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libcrypto_static_intermediates/libcrypto_static.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libuv_intermediates/libuv.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libcares_intermediates/libcares.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libdebugger_agent_intermediates/libdebugger_agent.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libchrome_zlib_intermediates/libchrome_zlib.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj/STATIC_LIBRARIES/libhttp_parser_intermediates/libhttp_parser.a external/webserver/jni/armeabi/node64
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libnodev8_intermediates/libnodev8_32.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libnode_intermediates/libnode.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libssl_static_intermediates/libssl_static.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libcrypto_static_intermediates/libcrypto_static.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libuv_intermediates/libuv.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libcares_intermediates/libcares.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libdebugger_agent_intermediates/libdebugger_agent.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libchrome_zlib_intermediates/libchrome_zlib.a external/webserver/jni/armeabi/node32
cp out/target/product/k53v1_64_op01/obj_arm/STATIC_LIBRARIES/libhttp_parser_intermediates/libhttp_parser.a external/webserver/jni/armeabi/node32

6. Finished. Go to build external webserver.