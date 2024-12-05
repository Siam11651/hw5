# HW5

## Task I

- **Set Port:** Define an int variable as default port and then parse argument using the code provided by your teacher. If there is any argument after -p flag we overwrite the default port;

- **SIGHUP:** Setup sigaction handler using SIGHUP signal following the given snippet
```c
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

#include "pbx.h"
#include "server.h"
#include "debug.h"

#include "string.h"
#include "sys/socket.h"
#include "sys/types.h"
#include "netinet/in.h"

static void terminate(int status);

void sighup_handler(int signal){
    terminate(EXIT_FAILURE);
}

/*
 * "PBX" telephone exchange simulation.
 *
 * Usage: pbx <port>
 */
int main(int argc, char* argv[]){
    // Option processing should be performed here.
    // Option '-p <port>' is required in order to specify the port number
    // on which the server should listen.
    int port = 8080;
    if(argc < 3 && strcmp(argv[1], "-p") != 0){
        terminate(1);
        fprintf(stderr, "Usage: %s -p <port>", argv[0]);
    }

    if((port = atoi(argv[2])) <= 0){
        terminate(1);
        fprintf(stderr, "Usage: Port number must be a non-negative numerical value");
    }

    struct sigaction sa = {.sa_handler = sighup_handler, .sa_flags = 0};
    sigemptyset(&sa.sa_mask);

    if (sigaction(SIGHUP, &sa, NULL) == -1) {
        exit(-1);
    }

    // Perform required initialization of the PBX module.
    debug("Initializing PBX...");
    pbx = pbx_init();

    // TODO: Set up the server socket and enter a loop to accept connections
    // on this socket.  For each connection, a thread should be started to
    // run function pbx_client_service().  In addition, you should install
    // a SIGHUP handler, so that receipt of SIGHUP will perform a clean
    // shutdown of the server.

    int server_socket = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in server_address = {.sin_family = AF_INET, .sin_port = htons(port), .sin_addr.s_addr = INADDR_ANY};

    if(bind(server_socket, (struct sockaddr*) &server_address, sizeof(server_address)) != 0){
        fprintf(stderr, "Error binding at port %d", port);
        close(server_socket);
    }

    if(listen(server_socket, 0) != 0){
        fprintf(stderr, "Error listening at port %d", port);
        close(server_socket);
    }

    while (1) {
        int client_fd = accept(server_socket, NULL, NULL);
        if (client_fd < 0) {
            perror("accept");
            continue; // Non-critical error, continue accepting new clients
        }

        pthread_t thread;
        if (pthread_create(&thread, NULL, (void* (*)(void*))pbx_client_service, (void*)(intptr_t)client_fd) != 0) {
            perror("pthread_create");
            close(client_fd);
        } else {
            pthread_detach(thread); // Detach thread to avoid memory leaks
        }
    }

    // Cleanup (this code won't actually execute unless you break out of the loop)
    close(server_socket);
    terminate(EXIT_SUCCESS);
}

/*
 * Function called to cleanly shut down the server.
 */
static void terminate(int status) {
    debug("Shutting down PBX...");
    pbx_shutdown(pbx);
    debug("PBX server terminating");
    exit(status);
}
```

## Task II

- **Initialize TU:** Use ```tu_init``` function to initialise new TU
- **Register to PBX:** Call ```pbx_init``` function to register pbx, we keep the scheme by demon program for now which is to use file descriptor as extension, we think about it later
- **TU Listener:** For now keep an empty for loop ```for(;;) {}``` block. In the for loop we keep reading the bytes one by one and store them in a buffer. We need to allocate a fixed sized buffer first, we may keep it 20 bytes.
- **Parse Command:** We keep a variable, maybe name it ```parse_state```, set it to 0, when we encounter a '\r' byte, we set it to 1, when it is 1, if we encounter a '\n' we then consider a command to be finished, we parse it then reset ```parse_state``` to 0.
- **Parse Command Words:** We first ```strncmp``` our command with "pickup", if matched we send response by writing "DIAL TONE" using ```write``` system call. But we need state management to handle all responses properly.
- **State Management:** We determine the state of TU using an int ```state```.
  - 0: On Hook
  - 1: Ringing
  - 2: Dial Tone
  - 3: Ring Back
  - 4: Busy Signal
  - 5: Connected
  - 6: Error

  First inside the infinite for loop, we check which state we are in using switch/case statement
  ```cpp
  switch(state) {
    case 0:
      // do on hook stuffs
      break;
    case 1:
      // do ringing stuffs
      break;
    case 2:
      // do dial tone stuffs
      break;
    case 3:
      // do ring back stuffs
      break;
    case 4:
      // do busy signal stuffs
      break;
    case 5:
      // do connected stuffs
      break;
    default:
      // do error stuff
  }
  ```