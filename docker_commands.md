# Docker cheatsheet/useful commands
### These are notes from various docker resources and trainings I took.

`docker build` -- creating an image
`docker push` -- pubishing
`docker run` -- run an image


### `docker run`
`docker run -it --rm bash` -- runs an interactive conatiner with the "bash" image. Note that "bash" is the name of the image and NOT the command we're executing

`docker run [--flags] IMAGE_NAME [CMD]` --> `docker run -it --rm bash echo 'hi'`

Common flags for `docker run`:
1. `-d` or `--detach` -- container continues to run under the daemon, in the background
2. `-e` or `--env=` -- sets an environment variable. `docker run -e FOO=bar bash env`.
3. `-i` or `-t` or `--interactive` -- run interactive ith terminal
4. `--rm` removes the container after the command finishes
5. `-p` or `--publish` -- forward ports to the container. `docker run -p 8000:80 -d IMAGE_NAME`
6. `--name` -- names the running container. `docker run -d --name=sleepy bash sleep 60`, `docker ps`, `docker kill sleepy`.

```bash
$ docker run -d bash sleep 10
e865b7dff52c...
$ docker ps
CONTAINER_ID IMAGE COMMAND ... NAMES
e865b7dff52c bash "docker-entrypoint.s…" ... mystifying_khayyam
```
10 seconds later:
```bash
$ docker ps
CONTAINER_ID IMAGE COMMAND ... NAMES
```

```bash
$ docker run -e FOO=bar bash env
...
FOO=bar
```
(the env command just prints all environment variables)

```bash
$ docker run -it bash
bash-5.0# _
```

```bash
$ docker run -p 8000:80 -d superorbital/color
8c92780c6f8e63fc872ba1...
$ docker ps
CONTAINER ID IMAGE ... PORTS ...
8c92780c6f8e superorbital/color ... 0.0.0.0:8000->80/tcp ...
$ http localhost:8000/version
{ "version": "31" }
```

`docker exec` -- runs a command in a running container!. Containers are NOT VMs! :)

```bash
$ docker run -d superorbital/color
de4615e4f840...
$ docker ps
CONTAINER ID IMAGE ... NAMES
de4615e4f840 superorbital/color ... elegant_blackwell
$ docker exec -it elegant_blackwell /bin/sh
# ps -e
PID TTY CMD
1 ? tini
6 ? ruby
15 pts/0 sh
21 pts/0 ps
```


`docker run -d IMAGE_NAME` --> `docker exec -it CONTAINER_NAME /bin/sh`. We can get the container name from `docker ps` if we didn't assign a name when running it. Note that bin/sh may not work if we don't have it available on the container (more on this later).

**`docker exec` launches a porocess in all of the same namespaces as the target container.**

`docker attach` -- attach local standard input, output and error streams to a running container.

```bash
$ docker ps
CONTAINER ID IMAGE ... NAMES
de4615e4f840 superorbital/color ... elegant_blackwell
$ docker attach elegant_blackwell
```
This will give no output..what's going on?
```bash
^C
[2019-06-07 23:01:17] FATAL Interrupt:
/usr/local/lib/ruby/2.5.0/webrick/server.rb:170:in `select'
...
./color:135:in `<main>'
[2019-06-07 23:01:17] INFO going to shutdown ...
$ docker ps
CONTAINER ID IMAGE ... NAMES
```


**`exec` runs a new process in the same namespaces, `attach` attaches your terminal to the existing process, which is useful mostly for runtime languages that have built-in debuggers such as NodeJS.**

`docker logs` -- In Docker, logs are just whatever we write to STDOUT! 

Common flags for `docker logs`:
1. `-f` or `--follow` -- follow the log output
2. `-t` or `--timestamps` -- include timestamps
3. `--tail N` -- number of lines to show
4. `--until TIMESTAMP` -- show logs before a time
5. `--since TIMESTAMP` -- show logs since time

`docker run -d --name loud ubuntu bash -c "while true; do date; sleep 1; done"` --> `docker logs --tail 3 loud` --> `docker logs -tf loud`

```bash
$ docker run -d --name loud ubuntu bash -c "while true; do date; sleep 1; done"
610004bdf09391d3f0e837263c1a2f94bd...
$ docker logs --tail 3 loud
Fri Jun 7 23:41:33 UTC 2019
Fri Jun 7 23:41:34 UTC 2019
Fri Jun 7 23:41:35 UTC 2019
$ docker logs -tf loud
...
2019-06-07T23:52:44.385090113Z Fri Jun 7 23:52:44 UTC 2019
2019-06-07T23:52:45.386918143Z Fri Jun 7 23:52:45 UTC 2019
2019-06-07T23:52:46.388482583Z Fri Jun 7 23:52:46 UTC 2019
^C
```

`docker ps` -- list containers. Shows only running containers by default!

Common flags for `docker ps`:
1. `-a` or `-all` -- show stopped containers as well
2. `-l` or `--latest` -- show the most recent container
3. `-q` or `--quiet` -- only display numeric IDs
4. `--no-trunc` -- doesn't truncate output.

```bash
$ docker run --name=id bash id
uid=0(root) gid=0(root) groups=0(root)...
$ docker run --name=nginx -d nginx
0e12abd6b3f72d89e29a1930a013bee3064298...
$ docker ps
CONTAINER ID IMAGE ... STATUS ... NAMES
0e12abd6b3f7 nginx ... Up 4 seconds ... nginx
$ docker ps -a
CONTAINER ID IMAGE ... STATUS ... NAMES
0e12abd6b3f7 nginx ... Up 14 seconds ... nginx
64be8f058839 bash ... Exited (0) 40 seconds ago ... id
$ docker ps --quiet
0e12abd6b3f7
$ docker ps -aq
0e12abd6b3f7
64be8f058839
```


`docker ps` == `docker container ps` == `docker container ls` == `docker container list`...don't ask me why


`docker kill` -- sends a signal to running containers.

```bash
$ docker run -d --name catcher superorbital/signal-catcher
aa8c5c649afd11634d85b5cb2f4cc5a92844229f245c0cd11007fec3bdd53355
$ docker logs --follow catcher &
[2019-06-08 20:27:05] INFO WEBrick::HTTPServer#start: pid=1 port=8000
...
$ docker kill --signal=HUP catcher
Caught a HUP signal. Ignoring.
$ docker kill --signal=USR1 catcher
Caught a USR1 signal. Ignoring.
$ docker kill --signal=TERM catcher
Caught a TERM signal. Usually that would kill me, but I'm stubborn. Try SIGKILL.
$ docker kill catcher
[1]+ Done docker logs --follow catcher
```

`docker kill` == `kill -9 ( SIGKILL )`..which is fine for local development. A more graceful way we can try `docker stop catcher`

```bash
$ docker stop catcher
Caught a TERM signal. Usually that would kill me, but I'm stubborn. Try SIGKILL.
(...10 seconds later...)
[1]+ Done docker logs --follow catcher
```

The main process inside the container will receive `SIGTERM` and after a grade period, `SIGKILL`.

`docker rm` -- remove stopped containers.

```bash
$ docker run -d --name sleepy bash sleep 1000
754a75aecf6b6b6c3fc5340e8ecce70d8d7311d6f05c72928c7a0f5725f232eb
$ docker run --name old bash sleep 1
$ docker ps -a
CONTAINER ID ... STATUS ... NAMES
c3fb77ae2ba1 ... Exited (0) 19 seconds ago ... old
754a75aecf6b ... Up 34 seconds ... sleepy
$ docker rm old
$ docker ps -a
CONTAINER ID ... STATUS ... NAMES
754a75aecf6b ... Up 48 seconds ... sleepy
```

`docker rm --force` -- kills a running container and removes it.

```bash
$ docker ps -a
CONTAINER ID ... STATUS ... NAMES
754a75aecf6b ... Up 48 seconds ... sleepy
$ docker rm sleepy
Error response from daemon:
You cannot remove a running container
Stop the container before attempting
removal or force remove
$ docker rm --force sleepy
$ docker ps -a
...nadda...
```

`docker rm --force == kill -9 ( SIGKILL )`. We used `docker run --rm` above, which essentially runs the remove command once the container is stopped!


`docker cp` -- copy files into and out of a container (running or stopped).

From the container:
```bash
$ docker cp mycontainer:/path/to/file localfile
$ docker cp mycontainer:/path/to/dir localdir
```

To the container:
```bash
$ docker cp localfile mycontainer:/path/to/file
$ docker cp localdir mycontainer:/path/to/dir
```

```bash
$ cat hello.txt
world
$ docker run -d --name sleepy bash sleep 1000
$ docker cp hello.txt sleepy:/tmp/hello.txt
$ docker exec sleepy cat /tmp/hello.txt
world
```

Works for stopped container, too:
```bash
$ docker kill sleepy
$ docker cp sleepy:/tmp/hello.txt new-local-file.txt
$ cat new-local-file.txt
world
```

When we `rm` a container, the files are deleted too:
```bash
$ docker rm sleepy
$ docker cp sleepy:/tmp/hello.txt new-local-file.txt
Error: No such container:path: sleepy:/tmp/hello.txt
```

To/from STDOUT:
```bash
$ foo | docker cp - mycontainer:/path/to/file
$ docker cp mycontainer:/path/to/dir - | bar
```

However, please note that STDOUT copy doesn't work intuitively. If `-` is specified for either SRC_PATH/DEST_PATH, the output is zipped/tarred!

```bash
$ docker cp mycontainer:/etc/hello.txt -
hello.txt0100666000175200017550000000000613477577635011202
0ustar0000000000000000world
```

