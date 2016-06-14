### Using BPF Tools: Node and JVM USDT Probes

In this lab, you will experiment with discovering USDT probes in a running process or an on-disk executable, enabling these probes at runtime, and collecting them using BPF's USDT support in the `argdist` and `trace` tools.

- - -

#### Task 1: Discovering USDT Probes

The `tplist` tool from the BCC collection can discover USDT probes in an on-disk executable, library, or running process. It can also tell you a lot of details about the probe locations and arguments. Unfortunately, it doesn't tell you the *meaning* of these arguments -- you often have to discover them yourself by using the documentation. In this section, you will walk through this process for the Node.js main executable, `node`.

If you haven't already, clone the [Node.js](https://github.com/nodejs/node) repository and build it with the `--with-dtrace` flag, which enables USDT support, using the following commands:

```
$ git clone https://github.com/nodejs/node.git
$ cd node
$ git checkout v6.2.1   # or whatever version is currently stable
$ ./configure --prefix=/opt/node --with-dtrace
$ make -j 3
$ sudo make install
```

Now, you can use `tplist` to discover USDT probes in the Node executable. Run the following command (provide the full paths to `tplist` and `node` as necessary):

```
$ tplist.py -l node 
```

This should print out a number of USDT probes, including `node:http__server__request` and `node:gc__start`. There are more probes embedded in a typical Node process, however -- you can see exactly what's available (possibly through additional libraries) by running `node` in the background and then using the following command:

```
$ tplist.py -p $(pidof node)
```

You should now see USDT probes from libc and libstdcxx as well. You can also ask `tplist` to print out more details about each probe. For example, here are the arguments for the `node:http__server_request` probe as reported by `tplist`:

```
$ tplist.py -l node -v '*server__request'
/home/vagrant/node/out/Release/node node:http__server__request [sema 0x1606c34]
  location 0xef4854 raw args: 8@%r14 8@%rax 8@-4328(%rbp) -4@-4332(%rbp) 8@-4288(%rbp) 8@-4296(%rbp) -4@-4336(%rbp)
    8 unsigned bytes @ register %r14
    8 unsigned bytes @ register %rax
    8 unsigned bytes @ -4328(%rbp)
    4   signed bytes @ -4332(%rbp)
    8 unsigned bytes @ -4288(%rbp)
    8 unsigned bytes @ -4296(%rbp)
    4   signed bytes @ -4336(%rbp)
```

To figure out what these probes and their arguments mean, we need to inspect the documentation or some authoritative source. In many cases, software instrumented with USDT probes will also ship with a set of files intended for use with SystemTap or DTrace, which can consume these probes. Sure enough, there's a [node.stp](https://github.com/nodejs/node/blob/master/src/node.stp) file that describes these events. Find the description for the
`node:http__server_request` probe:

```
probe node_http_server_request = process("node").mark("http__server__request")
{
    remote = user_string($arg3);
    port = $arg4;
    method = user_string($arg5);
    url = user_string($arg6);
    fd = $arg7;

    probestr = sprintf("%s(remote=%s, port=%d, method=%s, url=%s, fd=%d)",
        $$name, remote, port, method, url, fd);
}
```

It is quite clear what the arguments mean now, and we can use them with the other tracing tools that can retrieve them.

- - -

#### Task 2: Enabling and Tracing Node USDT Probes with `trace`

Let's now make sure that we can trace some HTTP requests with the USDT support embedded in Node. For this experiment, you can build your own Node HTTP server, or just use the [server.js](server.js) file, reproduced here in its entirety:

```javascript
var http = require('http');
var server = http.createServer(function (req, res) {
    res.end('Hello, world!');
});
server.listen(8080);
```

You can run this server using the following command:

```
$ node server.js
```

In a root shell, navigate to the **tools** directory under the BCC source, and use the following command to attach to the `node:http__server__request` USDT event and trace out a message whenever a request arrives. Note that the process id is required because the Node USDT events are not enabled by default. The tracing program must poke a global variable that the probing code reads in order to enable the probe, and that happens on a per-process basis.

```
# trace.py -p $(pidof node) 'u:/opt/node/node:http__server__request "%s %s", arg5, arg6'
```

Here, `arg5` is going to be the HTTP method and `arg6` is going to be the request URL -- as we discovered by inspecting the .stp file. Finally, make some HTTP requests to your server and make sure you see them in the trace:

```
$ curl localhost:8080
$ curl localhost:8080/index.html
$ curl 'localhost:8080/login?user=dave&pwd=123'
```

- - -

#### Task 3: Enabling and Tracing JVM USDT Probes with `argdist`

OpenJDK is also instrumented with a number of USDT probes. Most of them are enabled out of the box, and some must be enabled with a special `-XX:+ExtendedDTraceProbes` flag, because they incur a performance penalty. To explore some of these probes, navigate to your `$JAVA_HOME` and take a look in the **tapset** directory:

```
$ cd /etc/alternatives/java_sdk
$ ls tapset/
hotspot-1.8.0.77-1.b03.fc22.x86_64.stp
hotspot_gc-1.8.0.77-1.b03.fc22.x86_64.stp
hotspot_jni-1.8.0.77-1.b03.fc22.x86_64.stp
jstack-1.8.0.77-1.b03.fc22.x86_64.stp
```

These .stp files contain descriptions and declarations for a bunch of probes, including their arguments. As an example, try to find the `gc_collect_tenured_begin` and `gc_collect_tenured_end` probe descriptions.

TODO

- - -

#### Bonus: Discovering Probes in Oracle JDK

**TODO**

- - -
