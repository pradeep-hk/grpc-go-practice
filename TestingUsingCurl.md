**Create binary payload:**

1.Create text file(GreetRequest.txt) with payload:

    greeting : {
	     first_name: "Pradeep"
	     last_name: "HK"
    }
       
2.Encode to create gRPC payload:

    #!/bin/sh
    #
    # Steps:
    # - Encode using protoc command to get the binary data
    # - Take hexdump of the binary
    # - prefix it with compression-algorithm (00 for no compression) and message-length (in hexadecimal and in 4 bytes)
    # - convert back to binary    

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
    #
    # Steps:
    # - Take hexdump of the binary
    # - The first byte represents compression-algorithm (00 for no compression) 
    #   and next 4 bytes represents the message-length (in hexadecimal)
    #   So, the message starts from 11th octet
    # - Convert the message portion of hexdump to binary
    # - Decode using protoc

    PROTO_CMD="protoc -I ../../greetpb"
    OUTPUT_MSG_TYPE="greet.GreetResponse"
    PROTO_FILE="greet.proto"


    HEX_DUMP_MSG=`xxd -p -c 256 ./resp.bin | awk '{print substr($1,11)}'`
    echo $HEX_DUMP_MSG  | xxd -r -p | $PROTO_CMD --decode=$OUTPUT_MSG_TYPE $PROTO_FILE
   


   ---------------------------
    Sample decoded response:
    ---------------------------
    result: "Hello Pradeep"
    
   ---------------------------

**Testing GreetEveryone:**  

A.Create payload: 

Refer to steps (1) and (2) above and create 2 individual payloads - prad_GreetRequest.payload and then ravi_GreetRequest.payload
from the corresponding text payloads (remember to use INPUT_MSG_TYPE="greet.GreetEveryoneRequest")
Merge the 2 payloads to mimic streaming input

    cat prad_GreetRequest.txt
    greeting : {
	     first_name: "Pradeep"
	     last_name: "HK"
    }
    
    cat prad_GreetRequest.txt
    greeting : {
	     first_name: "Raviraj"
	     last_name: "DM"
    }
    
    dd if=prad_GreetRequest.payload >  GreetEveryoneRequest.payload  
    dd if=ravi_GreetRequest.payload >> GreetEveryoneRequest.payload
    
B.Use curl inside docker to execute gRPC calls

    curl --http2-prior-knowledge -X POST \
         -H "Content-Type: application/grpc" \
         -H "TE: trailers" \
         --data-binary @GreetEveryoneRequest.payload \
         http://<IP>:<PORT>/greet.GreetService/GreetEveryone \
         -o resp.bin --trace-ascii dump.txt

C.Decode the response


    xxd -p -c 256 ./resp.bin
    00000000110a0f48656c6c6f2050726164656570212000000000110a0f48656c6c6f205261766972616a2120 
      
    Note:
    - first byte (ie 00) represents the compression-algorithm
    - next 4 bytes (ie 00000011) represents the message-length
      11 in decimal is 17 and 17x2=34 octets
      Split the response based on this ie
      00000000110a0f48656c6c6f20507261646565702120   00000000110a0f48656c6c6f205261766972616a2120  
      
    PROTO_CMD="protoc -I ../../greetpb"
    OUTPUT_MSG_TYPE="greet.GreetEveryoneResponse"
    PROTO_FILE="greet.proto"      
      
    HEX_DUMP_RESP=00000000110a0f48656c6c6f20507261646565702120
    HEX_DUMP_MSG=`echo $HEX_DUMP_RESP | awk '{print substr($1,11)}'`
    echo $HEX_DUMP_MSG  | xxd -r -p | $PROTO_CMD --decode=$OUTPUT_MSG_TYPE $PROTO_FILE   
    result: "Hello Pradeep! "

    HEX_DUMP_RESP=00000000110a0f48656c6c6f205261766972616a2120
    HEX_DUMP_MSG=`echo $HEX_DUMP_RESP | awk '{print substr($1,11)}'`
    echo $HEX_DUMP_MSG  | xxd -r -p | $PROTO_CMD --decode=$OUTPUT_MSG_TYPE $PROTO_FILE 
    result: "Hello Raviraj! "  
         
