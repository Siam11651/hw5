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

## Task III

- **PBX Init:** We implement struct PBX as a linked list of pointer to struct TU.
```cpp
typedef struct pbx PBX {
  TU *tu;
  PBX *next;
};
```
- **PBX Register:** We append a new element to the linked list. We lock a mutex at the beginning of each register task and unlock it before returning the function.
- **PBX Unregister:** We iterate through the whole linked list to find the element whose TU pointer matches to that of parameter, detach it from linked list, connect the next item to previous item (basically remove an item from linked list) and free the pointer. We lock the same mutex at the beginning of each unregister task and unlock it before returning the function.
- **PBX Shutdown:** Iterate through all TU items in linked list and free all of them. We use the same mutex in the same manner here as well. To unregister TU, we basically call ```tu_unref``` on that TU pointer. On each TU pointer in the PBX linked list we iterating over.
- **PBX Dial:** Find the pointer to TU iterating over the linked list which matches the extension and send dial command through the file descriptor in TU. We lock the mutex here as well, because someone might dial to a TU and that TU may request to be unregistered concurrently.
### pbx.c
```c
/*
 * PBX: simulates a Private Branch Exchange.
 */
#include <stdlib.h>

#include "pbx.h"
#include "debug.h"
#include "pthread.h"
#include "unistd.h"
#include "tu.h"
#include "string.h"
#include "sys/socket.h"

typedef struct pbx_node{
    TU *tu;
    int ext;
    struct pbx_node *next;
} PBX_NODE;

struct pbx{
    PBX_NODE *head;
    pthread_mutex_t mutex;
};

void printPBX(PBX *pbx) {
    pthread_mutex_lock(&pbx->mutex);

    printf("PBX State:\n");
    PBX_NODE *current = pbx->head;
    if (!current) {
        printf("  No TUs registered.\n");
    } else {
        while (current) {
            int ext = current->ext;
            TU_STATE state = tu_get_state(current->tu);
            printf("  Extension %d: State = %s\n", ext, tu_state_names[state]);
            current = current->next;
        }
    }

    pthread_mutex_unlock(&pbx->mutex);
}

TU* pbx_get_target(PBX* pbx, int targetExt) {
    pthread_mutex_lock(&pbx->mutex);

    PBX_NODE* current = pbx->head;
    while (current != NULL) {
        if (current->ext == targetExt) {
            tu_ref(current->tu, "retrieving target TU");
            pthread_mutex_unlock(&pbx->mutex);
            return current->tu;
        }
        current = current->next;
    }

    pthread_mutex_unlock(&pbx->mutex);
    return NULL;
}

/*
 * Initialize a new PBX.
 *
 * @return the newly initialized PBX, or NULL if initialization fails.
 */
PBX *pbx_init() {
    // TO BE IMPLEMENTED
    PBX *pbx = malloc(sizeof(PBX));
    if(pbx == NULL){
        return NULL;
    }
    pbx->head = NULL;
    pthread_mutex_init(&pbx->mutex, NULL);
    return pbx;
}

/*
 * Shut down a pbx, shutting down all network connections, waiting for all server
 * threads to terminate, and freeing all associated resources.
 * If there are any registered extensions, the associated network connections are
 * shut down, which will cause the server threads to terminate.
 * Once all the server threads have terminated, any remaining resources associated
 * with the PBX are freed.  The PBX object itself is freed, and should not be used again.
 *
 * @param pbx  The PBX to be shut down.
 */
void pbx_shutdown(PBX *pbx) {
    // TO BE IMPLEMENTED
    pthread_mutex_lock(&pbx->mutex);

    PBX_NODE* current = pbx->head;
    while(current != NULL){
        shutdown(tu_fileno(current->tu), SHUT_RDWR);
        tu_unref(current->tu, "shutting everything");
        PBX_NODE *tmp = current;
        current = current->next;
        free(tmp);
    }

    pthread_mutex_unlock(&pbx->mutex);
    pthread_mutex_destroy(&pbx->mutex);
    free(pbx);
}

/*
 * Register a telephone unit with a PBX at a specified extension number.
 * This amounts to "plugging a telephone unit into the PBX".
 * The TU is initialized to the TU_ON_HOOK state.
 * The reference count of the TU is increased and the PBX retains this reference
 *for as long as the TU remains registered.
 * A notification of the assigned extension number is sent to the underlying network
 * client.
 *
 * @param pbx  The PBX registry.
 * @param tu  The TU to be registered.
 * @param ext  The extension number on which the TU is to be registered.
 * @return 0 if registration succeeds, otherwise -1.
 */
int pbx_register(PBX *pbx, TU *tu, int ext) {
    pthread_mutex_lock(&pbx->mutex);

    PBX_NODE *current = pbx->head;
    while (current != NULL) {
        if (current->ext == ext) {
            pthread_mutex_unlock(&pbx->mutex);
            fprintf(stderr, "Client already exists. Cannot register!\n");
            return -1;
        }
        current = current->next;
    }

    PBX_NODE *newNode = malloc(sizeof(PBX_NODE));
    if (newNode == NULL) {
        pthread_mutex_unlock(&pbx->mutex);
        return -1;
    }

    newNode->tu = tu;
    newNode->ext = ext;
    newNode->next = pbx->head;
    pbx->head = newNode;

    tu_ref(tu, "registration");
    tu_set_extension(tu, ext);

    pthread_mutex_unlock(&pbx->mutex);
    return 0;
}

/*
 * Unregister a TU from a PBX.
 * This amounts to "unplugging a telephone unit from the PBX".
 * The TU is disassociated from its extension number.
 * Then a hangup operation is performed on the TU to cancel any
 * call that might be in progress.
 * Finally, the reference held by the PBX to the TU is released.
 *
 * @param pbx  The PBX.
 * @param tu  The TU to be unregistered.
 * @return 0 if unregistration succeeds, otherwise -1.
 */
int pbx_unregister(PBX *pbx, TU *tu) {
    // TO BE IMPLEMENTED
    pthread_mutex_lock(&pbx->mutex);

    PBX_NODE *current = pbx->head;
    PBX_NODE *prev = NULL;

    while(current != NULL){
        if(current->tu == tu){
            tu_hangup(tu);

            if(prev){
                prev->next = current->next;
            }
            else{
                pbx->head = current->next;
            }

            tu_unref(tu, "unregistration");
            free(current);
            pthread_mutex_unlock(&pbx-> mutex);
            return 0;
        }

        prev = current;
        current = current->next;
    }

    pthread_mutex_unlock(&pbx->mutex);
    fprintf(stderr, "TU not found in PBX");
    return -1;
}

/*
 * Use the PBX to initiate a call from a specified TU to a specified extension.
 *
 * @param pbx  The PBX registry.
 * @param tu  The TU that is initiating the call.
 * @param ext  The extension number to be called.
 * @return 0 if dialing succeeds, otherwise -1.
 */

int pbx_dial(PBX *pbx, TU *tu, int ext) {
    // TO BE IMPLEMENTED
    pthread_mutex_lock(&pbx->mutex);

    PBX_NODE* current = pbx->head;
    TU* target = NULL;

    while(current != NULL){
        if(current->ext == ext){
            target = current->tu;
            tu_ref(target, "dailing");
            break;
        }
        current = current->next;
    }

    int result = -1;
    if(target != NULL){
        result = tu_dial(tu, target);
        tu_unref(target, "dailing");
    }

    pthread_mutex_unlock(&pbx->mutex);
    return result;
}
```
