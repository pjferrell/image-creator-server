image-creator-server Documentation
==================================


Configuration
-------------

You can configure the following locally:

- the base image to use:
  
        BASE_IMAGE = "codersos/linux-image-creator"

API
---

- **POST /create**  
  Post a json in the following format
  - as request body to `/create`
  - as the specification argument `/create?specification=...`
  
  ```
  {
    "redirect" : "REDIECT-URL",
    "image" : "IMAGE-NAME",
    "commands" : [
      {
        "name" : "COMMAND-NAME",
        "command" : "COMMAND",
        "arguments" : ["ARGUMENT", ...]
      },
      ...
    ]
  }
  ```

  These have the following meaning:
  
  - `redirect` must be given. `REDIRECT-URL` is the url where the user
    gets redirected to once the build started. It gets an additional argument
    `status` which is appended like this:
     - `http://asd.asd/x/` becomes
       `http://asd.asd/x/?status=STATUSURL`
     - `http://asd.asd/x?asd=asd` becomes
       `http://asd.asd/x?asd=asd&status=STATUSURL`
    This is done leaving as much intact as possible.
    `STATUSURL` is replaced with the url of the build status,
    see **GET /status/ID**.
  - `image` can be given. If it is given, the given docker image will be used.
    The server may restrict which images accepts to use.
    `IMAGE-NAME` is the name of the docker image.
  - `commands` is a list of commands that should be executed on the
    linux image.
    For each command in the list, the `name` attribute MUST be given.
    Also, either the `url` or the `command` attribute MUST be given.
    - `COMMAND-NAME` is the name of the command for identification.
      Example: `Install Firefox`
    - `COMMAND` is the command to execute. It gets saved to
      `/tmp/command`, is made executable and is executed.
      Examples:
      - ```
        #!/bin/bash
        wget http://some.docu/ment.txt -O /home/ubuntu/
        ```
      - ```
        #!/usr/bin/python
        # doing some python stuff
        ```
    - `ARGUMENT` is part of a list of arguments to the `COMMAND` file.
    - `arguments` must be given.
  
  Example request:
  ```
  curl -H "Content-Type: application/json" -X POST -d '{"redirect":"http://localhost/","commands":[{"name":"build iso","command":"#!/bin/bash\n/toiso/command.sh -q\n","arguments":[]}]}' http://localhost:80/create
  ```
  
- **GET /status/ID**  
  The result of this GET request is a JSON like this:
  ```
  {
    "status" : "STATUS-CODE",
    "commands" : [
      {
        "name" : "COMMAND-NAME",
        "output" : "COMMAND-OUTPUT",
        "status" : "STATUS-CODE"
        "exitcode" : EXIT-CODE,
      },
      ...
    ],
    "download" : "DOWNLOAD-URL"
  }
  ```
  The parts have the following meaning:
  - `STATUS-CODE` is one of the following:
    - `waiting` - if the process is not yet started
    - `running` - if the process is currently runnning
    - `stopped` - if the process succeeded
  - `commands` are a list of commands.
    All of the commands in the **POST /create** MUST be present.
    There MAY be additional commands.
    - `COMMAND-NAME`, see above, the name of a command.
    - `COMMAND-OUTPUT` - the stdout and stderr combined string of command
      output. This is useful for debugging.
      Commands with status `stopped` must have the `output` attribute.
    - `EXIT-CODE` is the return code of the command.
      It can be assumed that `0` means success and everything else is failure.
      Commands with status `stopped` must have the `exitcode` attribute.
  - `DOWNLOAD-URL` is the URL where the result can be downloaded once the
    process exited with `STATUS-CODE` `stopped`.
  
  Example request:
  ```
  wget -qO- http://localhost/status/0 ; echo
  ```
- **GET /status**  
  The status of the server. To see whether it is there, whether it is building or something else.
  ```
  {
    "status" : "STATUS-INIDICATION",
    "priority" : PRIORITY
  }
  ```
  Where the following meaning is assigned:
  - `STATUS-INDICATION` is one of these:
    - `ready` to indicate that the server can create images
    - `busy` to indicate that the server does not want to create images
  - `PRIORITY` is an integer with a build priority which is used to choose this server among others.
    A higher priority means that the server shall be favored.
  
- **GET /source**  
  The result is a zip file with the current source code.

- **GET /test/create**  
  The same as **GET /create** but no commands will be issued.

- **GET /test/status**  
  The same as **GET /status**.

Image API
---------

The server can transform any docker image into an iso file as long as certain things are made certain:

1. The docker image creates the iso file itself. Possibly with the last command. That this is done is not the responsibility of the server but of the request.
2. To get the iso file, the server looks for and executes this code:

   ```
   /toiso/iso_path.sh
   ```

   Which outputs a path to the iso file without line break at the end, for example `/toiso/CodersOS.iso`.
   Here is an example file:
   
   ```
   #!/bin/bash
   echo -n "/toiso/CodersOS.iso"
   ```
   
   See the [here][toiso] for an example implementation.
   
   
 [toiso]: https://github.com/CodersOS/linux-iso-creator/blob/d7e66ba0922de31de37a012c81de8a1b5486de86/toiso/iso_path.sh
