# HW5

## Task I

- **Set Port:** Define an int variable as default port and then parse argument using the code provided by your teacher. If there is any argument after -p flag we overwrite the default port;

- **SIGHUP:** Setup sigaction handler using SIGHUP signal following the given snippet
```c
#include <stdlib.h>
#include <unistd.h>
#include <signal.h> // new header for this sub-task

#include "pbx.h"
#include "server.h"
#include "debug.h"

static void terminate(int status);

void sighup_handler(int signal) {
  // we may need to update it later, for task i, this seems enough
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

  // we allocate sighup handler before pbx_init
  struct sigaction sa = {sighup_handler, 0};

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

  fprintf(stderr, "You have to finish implementing main() "
    "before the PBX server will function.\n");

  terminate(EXIT_FAILURE);
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