**Create binary payload:**

1.Create text file(GreetRequest.txt) with payload:

    greeting : {
	     first_name: "Pradeep"
	     last_name: "HK"
    }
       
2.Encode to create gRPC payload:

    #!/bin/sh

    PROTO_CMD="protoc -I ../../greetpb"
    INPUT_MSG_TYPE="greet.GreetRequest"
    PROTO_FILE="greet.proto"

    FILE_NAME=GreetRequest

    $PROTO_CMD --encode=$INPUT_MSG_TYPE $PROTO_FILE < ./$FILE_NAME.txt > ./$FILE_NAME.bin

    MSG=`xxd -p -c 256 ./$FILE_NAME.bin`
    MSG_LEN=`echo ${#MSG}`
    BYTES_MSG_LEN=$((MSG_LEN/2))
    BYTES_MSG_LEN_HEX=`printf '%08x\n' $BYTES_MSG_LEN`
    COMPRESSION_ALG=00
    HEX_PAYLOAD="$COMPRESSION_ALG$BYTES_MSG_LEN_HEX$MSG"
    echo -n $HEX_PAYLOAD | xxd -r -p - ./$FILE_NAME.payload  
    
**Create ubuntu docker with curl having support for http2 :** 

    docker run --name ubuntu -dit ubuntu bash
    docker exec -it ubuntu bash
    
    root@9fd99132d25e:/# apt-get update
    root@9fd99132d25e:/# apt-get install curl   
    
    
**Use curl inside docker to execute gRPC calls :** 

    root@9fd99132d25e:/# curl --http2-prior-knowledge  -k --raw -X POST \
    -H "Content-Type: application/grpc" \
    -H "TE: trailers" \
    --data-binary @GreetRequest.payload \
    http://<IP>:<PORT>/greet.GreetService/Greet \
    -o resp.bin --trace-ascii dump.txt    
    
**Decode gRPC response :**

    #!/bin/sh

    PROTO_CMD="protoc -I ../../greetpb"
    OUTPUT_MSG_TYPE="greet.GreetResponse"
    PROTO_FILE="greet.proto"


    HEX_DUMP_MSG=`xxd -p -c 256 ./resp.bin | awk '{print substr($1,11)}'`
    echo $HEX_DUMP_MSG  | xxd -r -p | $PROTO_CMD --decode=$OUTPUT_MSG_TYPE $PROTO_FILE
   


   ---------------------------
    Sample decoded response:
    ---------------------------
    result: "Hello Pradeep"
