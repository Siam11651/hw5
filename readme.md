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
#include "pthread.h"
#include "stdint.h"

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
    // pbx = pbx_init();

    // TODO: Set up the server socket and enter a loop to accept connections
    // on this socket.  For each connection, a thread should be started to
    // run function pbx_client_service().  In addition, you should install
    // a SIGHUP handler, so that receipt of SIGHUP will perform a clean
    // shutdown of the server.

    int server_socket = socket(AF_INET, SOCK_STREAM, 0);
    if(server_socket < 0){
        fprintf(stderr, "Error creating a socket");
        close(server_socket);
    }
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
        debug("Listening on port: %d", port);
        int client_fd = accept(server_socket, NULL, NULL);
        if (client_fd < 0) {
            perror("accept");
            continue;
        }

        int *client_fd_ptr = malloc(sizeof(int));
        if (client_fd_ptr == NULL) {
            perror("malloc failed");
            close(client_fd);
            continue;
        }
        *client_fd_ptr = client_fd;

        pthread_t thread;
        if (pthread_create(&thread, NULL, (void* (*)(void*))pbx_client_service, client_fd_ptr) != 0) {
            perror("pthread_create");
            free(client_fd_ptr);
            close(client_fd);
        } else {
            pthread_detach(thread);
        }
    }

    close(server_socket);
    terminate(EXIT_SUCCESS);

    // fprintf(stderr, "You have to finish implementing main() "
	//     "before the PBX server will function.\n");

    // terminate(EXIT_FAILURE);
    //terminate(EXIT_SUCCESS);
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
  **Server.c**
```c
/*
 * "PBX" server module.
 * Manages interaction with a client telephone unit (TU).
 */
#include <stdlib.h>
#include "debug.h"
#include "pbx.h"
#include "server.h"
#include "tu.h"
#include "string.h"


TU* pbx_get_target(PBX* pbx, int num){
    return tu_init(123); // TODO RETURNS RANDOM
}

void parse_command(char* buffer, TU* newTU, int client){
    if (strncmp(buffer, "pickup", 6) == 0) {
        tu_pickup(newTU);
    } else if (strncmp(buffer, "hangup", 6) == 0) {
        tu_hangup(newTU);
    } else if (strncmp(buffer, "dial", 4) == 0) {
        int number = atoi(buffer + 5);
        TU* target = pbx_get_target(pbx, number);
        if (target) {
            tu_dial(newTU, target);
            tu_unref(target, "dial command complete");
        } else {
            tu_dial(newTU, NULL);
        }
    } else if (strncmp(buffer, "chat", 4) == 0) {
        tu_chat(newTU, (buffer + 5));
    } else {
        write(client, "Invalid Command\r\n", 17);
    }
}

void mock_parse_command(char* buffer, int client) {
    if (strncmp(buffer, "pickup", 6) == 0) {
        debug("Mock: pickup command executed");
    } else if (strncmp(buffer, "hangup", 6) == 0) {
        debug("Mock: hangup command executed");
    } else if (strncmp(buffer, "dial", 4) == 0) {
        debug("Mock: dial command executed with number: %s", buffer + 5);
    } else if (strncmp(buffer, "chat", 4) == 0) {
        debug("Mock: chat command executed with message: %s", buffer + 5);
    } else {
        write(client, "Invalid Command\r\n", 17);
    }
}


/*
 * Thread function for the thread that handles interaction with a client TU.
 * This is called after a network connection has been made via the main server
 * thread and a new thread has been created to handle the connection.
 */

void *pbx_client_service(void *arg) {
    int client = *(int*)arg; // Extract file descriptor
    free(arg);
    debug("Connected to: %d", client);

    TU* newTU = tu_init(client); // Initialize the TU
    if (!newTU) {
        close(client);
        return NULL;
    }

    if (pbx_register(pbx, newTU, client) < 0) {
        tu_unref(newTU, "Failed to register TU");
        close(client);
        return NULL;
    }

    char buffer[20];
    int parse_state = 0;
    size_t buffer_pos = 0;

    for (;;) {
        char c;
        int byte_read = read(client, &c, 1);
        if (byte_read <= 0) {
            pbx_unregister(pbx, newTU);
            tu_unref(newTU, "Client disconnected");
            close(client);
            return NULL;
        }

        if (buffer_pos < sizeof(buffer) - 1) {
            buffer[buffer_pos++] = c;
        }

        if (c == '\r') {
            parse_state = 1;
        } else if (parse_state == 1 && c == '\n') {
            buffer[buffer_pos] = '\0';
            // parse_command(buffer, newTU, client);
            mock_parse_command(buffer, client);
            parse_state = 0;
            buffer_pos = 0;
        }
    }
}
```
