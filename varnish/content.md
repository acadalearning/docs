# What is Varnish?

Varnish is an HTTP accelerator designed for content-heavy dynamic web sites as well as APIs. In contrast to other web accelerators, such as Squid, which began life as a client-side cache, or Apache and nginx, which are primarily origin servers, Varnish was designed as an HTTP accelerator. Varnish is focused exclusively on HTTP, unlike other proxy servers that often support FTP, SMTP and other network protocols.

> [wikipedia.org/wiki/Varnish_(software)](https://en.wikipedia.org/wiki/Varnish_(software))

%%LOGO%%

# How to use this image.

## Basic usage

Create a `default.vcl` file:

```vcl
# specify the VCL syntax version to use
vcl 4.1;

# import vmod_dynamic for better backend name resolution
import dynamic;

# we won't use any static backend, but Varnish still need a default one
backend default none;

# set up a dynamic director
# for more info, see https://github.com/nigoroll/libvmod-dynamic/blob/master/src/vmod_dynamic.vcc
sub vcl_init {
        new d = dynamic.director(port = "80");
}

sub vcl_recv {
	# force the host header to match the backend (not all backends need it,
	# but example.com does)
	set req.http.host = "example.com";
	# set the backend
	set req.backend_hint = d.backend("example.com");
}
```

Then run:

```console
# we need the configuration file at /etc/varnish/default.vcl,
# our workdir has to be mounted as tmpfs to avoid disk I/O,
# and we'll use port 8080 to talk to our container (internally listening on 80)
$ docker run \
	-v /path/to/default.vcl:/etc/varnish/default.vcl:ro \
	--tmpfs /var/lib/varnish/varnishd:exec \
	-p 8080:80 \
	%%IMAGE%%
```

From there, you can visit `localhost:8080` in your browser and see the example.com homepage.

Alternatively, a simple `Dockerfile` can be used to generate a new image that includes the necessary `default.vcl` (which is a much cleaner solution than the bind mount above):

```dockerfile
FROM %%IMAGE%%

COPY default.vcl /etc/varnish/
```

Place this file in the same directory as your `default.vcl`, run `docker build -t my-varnish .`, then start your container:

```console
$ docker --tmpfs /var/lib/varnish/varnishd:exec -p 8080:80 my-varnish
```

## Reloading the configuration

The images all ship with [varnishreload](https://github.com/varnishcache/pkg-varnish-cache/blob/master/systemd/varnishreload#L42) which allows you to easily update the running configuration without restarting the container (and therefore losing your cache). At its most basic, you just need this:

```console
# update the default.vcl in your container
docker cp new_default.vcl running_container:/etc/varnish/default.vcl
# run varnishreload
docker exec running_container varnishreload
```

Note that `varnishreload` also supports reloading other files (it doesn't have to be `default.vcl`), labels (`l`), and garbage collection of old labeles (`-m`) among others. To know more, run

```console
docker run varnish varnishreload -h
```

## Additional configuration

By default, the containers will use a cache size of 100MB, which is usually a bit too small, but you can quickly set it through the `VARNISH_SIZE` environment variable:

```console
$ docker run --tmpfs /var/lib/varnish/varnishd:exec -p 8080:80 -e VARNISH_SIZE=2G %%IMAGE%%
```

Additionally, you can add arguments to `docker run` after `%%IMAGE%%`, if the first one starts with a `-`, they will be appendend to the [default command](https://github.com/varnish/docker-varnish/blob/master/docker-varnish-entrypoint#L8):

```console
# extend the default keep period
$ docker run --tmpfs /var/lib/varnish/varnishd:exec -p 8080:80 -e VARNISH_SIZE=2G %%IMAGE%% -p default_keep=300
```

If your first argument after `%%IMAGE%%` doesn't start with `-`, it will be interpreted as a command to override the default one:

```console
# show the command-line options
$ docker run %%IMAGE%% varnishd -?

# list parameters usable with -p
$ docker run %%IMAGE%% varnishd -x parameter

# run the server with your own parameters (don't forget -F to not daemonize)
$ docker run %%IMAGE%% varnishd -F -a :8080 -b 127.0.0.1:8181 -t 600 -p feature=+http2
```

## vmods (since 7.1)

As mentioned above, you can use [vmod_dynamic](https://github.com/nigoroll/libvmod-dynamic) for backend resolution. The [varnish-modules](https://github.com/varnish/varnish-modules) collection is also included in the image. All the documentation regarding usage and syntax can be found in the [src/](https://github.com/varnish/varnish-modules/tree/master/src) directory of the repository.
