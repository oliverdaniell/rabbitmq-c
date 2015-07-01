# vim:set ts=2 sw=2 sts=2 et:
include_directories(${LIBRABBITMQ_INCLUDE_DIRS})

if (WIN32)
    set(PLATFORM_DIR win32)
else (WIN32)
    set(PLATFORM_DIR unix)
endif (WIN32)

set(COMMON_SRCS
    utils.h
    utils.c
    ${PLATFORM_DIR}/platform_utils.c
    )
add_definitions(-DAMQP_STATIC)
add_executable(amqp_sendstring amqp_sendstring.c ${COMMON_SRCS})
target_link_libraries(amqp_sendstring rabbitmq-static)

add_executable(amqp_rpc_sendstring_client amqp_rpc_sendstring_client.c ${COMMON_SRCS})
target_link_libraries(amqp_rpc_sendstring_client rabbitmq-static)

add_executable(amqp_exchange_declare amqp_exchange_declare.c ${COMMON_SRCS})
target_link_libraries(amqp_exchange_declare rabbitmq-static)

add_executable(amqp_listen amqp_listen.c ${COMMON_SRCS})
target_link_libraries(amqp_listen rabbitmq-static)

add_executable(amqp_producer amqp_producer.c ${COMMON_SRCS})
target_link_libraries(amqp_producer rabbitmq-static)

add_executable(amqp_connect_timeout amqp_connect_timeout.c ${COMMON_SRCS})
target_link_libraries(amqp_connect_timeout rabbitmq-static)

add_executable(amqp_consumer amqp_consumer.c ${COMMON_SRCS})
target_link_libraries(amqp_consumer rabbitmq-static)

add_executable(amqp_unbind amqp_unbind.c ${COMMON_SRCS})
target_link_libraries(amqp_unbind rabbitmq-static)

add_executable(amqp_bind amqp_bind.c ${COMMON_SRCS})
target_link_libraries(amqp_bind rabbitmq-static)

add_executable(amqp_listenq amqp_listenq.c ${COMMON_SRCS})
target_link_libraries(amqp_listenq rabbitmq-static)

if (ENABLE_SSL_SUPPORT)
add_executable(amqps_connect_timeout amqps_connect_timeout.c ${COMMON_SRCS})
target_link_libraries(amqps_connect_timeout rabbitmq-static)

add_executable(amqps_sendstring amqps_sendstring.c ${COMMON_SRCS})
target_link_libraries(amqps_sendstring rabbitmq-static)

add_executable(amqps_exchange_declare amqps_exchange_declare.c ${COMMON_SRCS})
target_link_libraries(amqps_exchange_declare rabbitmq-static)

add_executable(amqps_listen amqps_listen.c ${COMMON_SRCS})
target_link_libraries(amqps_listen rabbitmq-static)

add_executable(amqps_producer amqps_producer.c ${COMMON_SRCS})
target_link_libraries(amqps_producer rabbitmq-static)

add_executable(amqps_consumer amqps_consumer.c ${COMMON_SRCS})
target_link_libraries(amqps_consumer rabbitmq-static)

add_executable(amqps_unbind amqps_unbind.c ${COMMON_SRCS})
target_link_libraries(amqps_unbind rabbitmq-static)

add_executable(amqps_bind amqps_bind.c ${COMMON_SRCS})
target_link_libraries(amqps_bind rabbitmq-static)

add_executable(amqps_listenq amqps_listenq.c ${COMMON_SRCS})
target_link_libraries(amqps_listenq rabbitmq-static)
endif (ENABLE_SSL_SUPPORT)
