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


## `docker build`
Recalld:
* Container: a set of processes running with isolated PIDs, network and filesystem. `docker run` creates containers from images.
* Image: the directories, files and metadata used to create a container. `docker build` creates images from Dockerfiles.


_A Dockerfile is a set of instructions that Docker executes
to create the image filesystem and record metadata
about containers created from it._

```bash
$ ls
total 12K
-rw-rw-rw- student Jun 12 22:12 Dockerfile
-rw-rw-rw- student Jun 12 22:12 app.py
```

`app.py`:
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

`Dockerfile`:
```Docker
FROM ubuntu:20.04
RUN apt-get update -y && \
apt-get install -y \
python3-pip \
python3-dev
RUN pip install flask==1.0.2
WORKDIR /app
COPY . .
CMD [ "python3", "./app.py" ]
```
Here's what's happening inside:
1. Start with a base Ubuntu image
2. Install Python
3. Install the Python packages
4. Create and change to /app
5. Add the application files
6. Set the default container command


Building the image:
```bash
$ docker build -t app .
```

```bash
Sending build context to Docker daemon 4.096kB
Step 1/6 : FROM ubuntu:20.04
20.04: Pulling from library/ubuntu
6abc03819f3e: Pull complete
05731e63f211: Pull complete
0bd67c50d6be: Pull complete
Digest: sha256:f08638ec7dd...
Status: Downloaded newer image for ubuntu:20.04
---> 7698f282e524
Step 2/6 : RUN apt-get update -y && apt-get install...
---> Running in 87cfbcf57d9b
Get:1 http://security.ubuntu.com/ubu...
...
Step 3/6 : RUN pip3 install flask==1.0.2
---> Running in 38f30c897e2f
Collecting flask==1.0.2
...
Successfully built df882d4c1567
Successfully tagged app:latest
```

```bash
$ docker run -d -p 5000:5000 --rm --name=app app
35239b3fa8384afff8b469b8bbe394768151...
$ http :5000
Hello World!
```

`docker build` -- builds an image from a Dockerfile
Common flags for `docker build`:
1. `-t` -- set the image name and tag
2. `-q` -- just print the final image ID
3. `--pull` -- force re-pulling of base imae
4. `--no-cache` -- force recreation of all layers

Basic dockerfile format:
```Docker
Comment
INSTRUCTION arguments
INSTRUCTION arguments \
more arguments \
on another line
...
```
(instructions are uppercase by convention)

Set the base image: `FROM <image>[:<tag>|@<digest>]` --> `FROM ubuntu:20.04`, `FROM ubuntu` (assumes :latest, bad), `FROM ubuntu:abcd1234` (great), `FROM scratch` (empty!).

*Every Dockerfile must start with `FROM`. If you don't want a filesystem, you can use `FROM scratch`.

`RUN <command>` -- runs a command in the builder container.

```bash
FROM ubuntu:20.04
RUN echo "world" > /hello
```
```bash
$ docker build -t example .
Step 1/2: FROM ubuntu:20.04
---> 7698f282e524
Step 2/2: RUN echo "world" > /hello
---> Running in 23a4b74f7d48
...
Successfully built a26fa865dec9
Successfully tagged example:latest
```
```bash
$ docker run example cat /hello
world
```

*Everything after `RUN` is the command; no need to quote or escape: `RUN echo "world" > /hello` (no need for this).

`RUN` commands can span multiple lines:
```Docker
RUN apt-get update -y && \
apt-get install -y \
python3-pip \
python3-dev
```
**Important: there is also an "exec-form" of the `RUN` command:**
`RUN ["cat", "/hello"]`

* Doesn't use a shell, so no environment variables or expansions
* Useful for `FROM scratch` containers where `/bin/sh` isn't available

`ENV` -- set environment variables.

```Docker
FROM ubuntu:20.04
ENV NAME=Bob
CMD echo Hello, $NAME
```
```bash
$ docker build -t example .
...
Successfully built 14d266eb689b
Successfully tagged example:latest
$ docker run example
Hello, Bob
$ docker run example env
...
NAME=Bob
...
```

__`ENV` variables are also set in your build context:__
```Docker
FROM ubuntu:20.04
ENV NAME=Bob
# In the build container
RUN echo $NAME > /name
```
```bash
$ docker build -t example .
Sending build context to Docker
Step 1/4 : FROM ubuntu:20.04
...
Successfully built 21b08e272d5f
Successfully tagged example:latest
$ docker run example cat /name
Bob
```

`WORKDIR <directory>` -- `cd` to directory for all future instructions.

```Docker
FROM ubuntu:20.04
WORKDIR /tmp
RUN echo world > hello
```
```bash
$ docker build -t app .
...
$ docker run app ls /tmp/hello
/tmp/hello
```

`WORKDIR` also creates the directory if it doesn't exist:
```Docker
FROM ubuntu:20.04
WORKDIR /foo/bar
RUN echo world > hello
```
```bash
$ docker build -t app .
...
$ docker run app ls /foo/bar/hello
/foo/bar/hello
```

This works not just for build context, but also for the final container:
```Docker
FROM ubuntu:20.04
WORKDIR /foo/bar
```
```bash
$ docker build -t image .
$ docker run image pwd
/foo/bar
$ docker run -it image /bin/bash
# pwd
/foo/bar
```

`WORKDIR` can be used mant times.

```Docker
FROM ubuntu:20.04
RUN apt-get update -y && apt-get install -y curl
WORKDIR /usr/local
ENV URL=https://dl.google.com/go/go1.12.6.linux-amd64.tar.gz
RUN curl -sSL $URL | tar -zxvf -
ENV PATH=/usr/local/go/bin:$PATH
WORKDIR /app
COPY myapp.go .
CMD ["go", "run", "myapp.go"]
```


`COPY <src> <dst>` -- copies files from the local directory to your image

```Docker
FROM ubuntu:20.04
RUN mkdir /app
COPY myapp.go /app
```

```bash
$ ls
total 8.0K
-rw-rw-r-- 1 Dockerfile
-rw-rw-r-- 1 myapp.go
$ docker build -t app .
...
Step 3/3 : COPY myapp.go /app
---> e0db64914d51
Successfully built e0db64914d51
Successfully tagged app:latest
$ docker run app ls /app/myapp.go
/app/myapp.go
```

`COPY` supports wildcards:
```bash
$ ls
total 8.0K
-rw-rw-r-- Dockerfile
-rw-rw-r-- helper.go
-rw-rw-r-- myapp.go
-rw-rw-r-- number-1.txt
-rw-rw-r-- number-2.txt
```

```Docker
FROM ubuntu:20.04
RUN mkdir /app
COPY *.go /app
COPY number-?.txt /app
```
```Docker
FROM ubuntu:20.04
RUN mkdir /app
COPY * /app
```

It can `COPY` all the files in a directory:
```Docker
FROM ubuntu:20.04
RUN mkdir /app
COPY . /app
```

It also respects `WORKDIR`:
```Docker
FROM ubuntu:20.04
WORKDIR /app
COPY . .
```

`<dst>` is auto created when copying:
```Docker
FROM ubuntu:20.04
RUN mkdir /app
COPY . /app
```
```Docker
FROM ubuntu:20.04

COPY . /app
```

However `COPY` cannot read files from outside the 'build context':
```Docker
FROM ubuntu:20.04
COPY /etc/passwd /app
```
```bash
$ docker build .
Sending build context to Docker daemon 17.41kB
...
Step 3/3 : COPY /etc/passwd /app
COPY failed: stat /var/lib/docker/tmp/.../etc/passwd: no such file or directory
```

Put simply (very simply), the *build context is the dorectory you pass to `docker build`*.

`CMD <command>` -- set the default command for new containers. It does nothing but set metadata.

```Docker
FROM python:3.8.3
CMD [ "python", "-c", "print('Hello from Python!')" ]
```
```bash
$ docker build -t app .
...
$ docker run app
Hello from Python!
```

```Docker
FROM python:3.8.3
CMD [ "python", "-c", "print('Hello!')" ]
```
```bash
$ docker run app
Hello from Python!
```
```bash
$ docker run app python -c "print('Goodbye')"
Goodbye
```

There are *two* forms of `CMD`:
1. Shell form: `CMD python -3 app.py`
    * More natural
    * Invoked via `sh -c`
    * Allows for ENV viariables (due to above)
    * However it's not running as PID1
    * `sh` hides signals...
2. Exec form: `CMD [ "python", "-3", "app.py" ]`
    * **not** run via `sh`
    * Runs as PID1
    * Gets all signals
    * Works with `FROM scratch`
    * **The preferred form!**

Some gotchas with exec form:
* Doesn't work:
```Docker
CMD [ "python",
"-c",
"print('Hello from Python!')" ]
```

* Works:
```Docker
CMD [ "python", \
"-c", \
"print('Hello from Python!')" ]
```

`CMD` vs. `RUN` -- what's the difference?
`RUN`: Runs commands in the build context, which change the filesystem, which becomes the image.
`CMD`: Records which command to run by default in any container created from the image.

`ENTRYPOINT` -- allows for docker images to be used as commands:

In the beginning, there was only `CMD`. But someone thought what if I could package all of my command line utilities as docker images???

```Docker
FROM ubuntu:20.04
RUN apt-get update && \
apt-get install -y \
ssh-client
CMD ["ssh"]
````
```bash
$ docker build -t ssh .
$ docker run ssh
usage: ssh [-46AaCfGgKkMNnq...
[-D [bind_addres...
[-F configfile] ...
[-J [user@]host[...
[-O ctl_cmd] [-o...
[-S ctl_path] [-...
[user@]hostname]...
```

```bash
$ docker run ssh -V
docker: Error ... "exec: "-V":
executable file not found in $PATH": unknown.
```
Can you identify the culprit? `ssh` in `docker run ssh -V` is the image name, not the `CMD`! When you add arguments to `docker run` , they _replace_ the `CMD`.

```bash
$ docker run ssh ssh -V
OpenSSH_8.2p1 Ubuntu-4ubuntu0.1, OpenSSL 1.1.1f 31 Mar 2020
```
Now this is awkward...enter`ENTRYPOINT`!

```Docker
from ubuntu:20.04
RUN apt-get update && apt-get install -y ssh-client
ENTRYPOINT ["ssh"]
```
```bash
$ docker run ssh
usage: ssh [-46AaCfGgKkMNnqsTtVvXxYy] [-b bind_address] ...
[-D [bind_address:]port] [-E log_file] [-e es...
[-F configfile] [-I pkcs11] [-i identity_file...
[-J [user@]host[:port]] [-L address] [-l logi...
[-O ctl_cmd] [-o option] [-p port] [-Q query_...
[-S ctl_path] [-W host:port] [-w local_tun[:r...
[user@]hostname [command]
$ docker run ssh -V
OpenSSH_8.2p1 Ubuntu-4ubuntu0.1, OpenSSL 1.1.1f 31 Mar 2020
```

`ENTRYPOINT` is prepended to any `CMD` or `docker run img command` arguments. If you use `ENTRYPOINT`, then `CMD` becomes _arguments_.

```Docker
from ubuntu:20.04
RUN apt-get update && apt-get install -y ssh-client
ENTRYPOINT ["ssh"]
CMD ["-V"]
```
```bash
$ docker run ssh
OpenSSH_8.2p1 Ubuntu-4ubuntu0.1, OpenSSL 1.1.1f 31 Mar 2020
```
```bash
$ docker run -it ssh user@someaddress
user@someaddress's password:
student:~ $
```

Some advice for usin `ENTRYPOINT`:
1. Don't.
2. Always use Exec-form for both `ENTRYPOINT` and `CMD`
3. Don't use `ENTRYPOINT`...

Review:
1. `docker build` build images
2. By running instructions from a Dockerfile
3. Like `FROM` , `RUN` , `ENV` , `WORKDIR` , `COPY` , `CMD`
4. One at a time
5. Inside a special build container
6. And saving the resulting image

Nifty shortcut for killing all running containers:
`docker kill $(docker ps -q)`