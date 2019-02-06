# Inc directory for Azure IoT Device SDK for C 
set(AZUREIOT_INC_FOLDER "/usr/include/azureiot" "/usr/include/azureiot/inc")
include_directories(${AZUREIOT_INC_FOLDER})


file(GLOB imagenetCameraSources *.cpp)
file(GLOB imagenetCameraIncludes *.h )

cuda_add_executable(imagenet-camera ${imagenetCameraSources})

# Add necessary libraries for Azure IoT Device SDK for C
target_link_libraries(imagenet-camera nvcaffe_parser nvinfer jetson-inference
    serializer
    iothub_client
    iothub_client_amqp_transport
    uamqp
    aziotsharedutil
    pthread
    curl
    ssl
    crypto
    uuid
    parson
    m)
