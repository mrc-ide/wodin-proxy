# wodin-proxy

Proxy configuration for wodin. This can bring up [nginx](https://nginx.org/) in either http or https mode, pointing at a number of wodin instances. Generally we'll assume that these are running each on a different container, but they could also be running on the same machine but with a series of ports.

You may want a directory, say `root/` containing files to be served for paths that do not match any sites (e.g., a generic index page, 404 and 500 error files), otherwise you'll get a set of default nginx pages.

Bringing things up in http mode is easy:

```
docker run -d --network wodin-nw --name proxy \
       -p 80:80 \
       -v $PWD/root:/wodin/root:ro \
       -e WODIN_SITE_DEMO=wodin-demo:3000 \
       mrcide/wodin-proxy:latest localhost \
       --site demo=wodin-demo:3000 \
```

Here, we use the `:ro` modifier on the root bind mount to emphasise that this is a readonly mount; the proxy will not modify anything here.

The `localhost` argument is the hostname that the site will serve on; for a http-only site for local testing, `localhost` is a good idea, but for nothing else (see below for https details). We need to expose only port 80 to the host here.

The last argument is a wodin site to proxy. There can be multiple of these arguments. The argument format is `<site>=<location>` where `site` is the path to serve the site below the proxy (see below). The `location` component is the `container:port` combination where the wodin instance is serving.

With this setup,

* http://localhost returns a generic index page
* http://localhost/demo and http://localhost/demo/ take you to the wodin demo site

Note that the wodin app needs to have the same base url as the proxy, so you'll bring up wodin like

```
docker run [...] mrcide/wodin:latest --base-url=http://localhost/demo/
```

Bringing things up in https mode requires that we:

* expose port 443
* use an externally meaningful hostname
* tell the proxy we want it to run in ssl mode
* also get the certificate and key into the container

The certificate must be valid for the hostname!

First, bring up the container (with the first three changes from above made):

```
docker run -d --network wodin-nw --name proxy \
       -p 80:80 -p 443:443 \
       -v $PWD/root:/wodin/root:ro \
       -v $PWD/sites:/wodin/sites:ro \
       mrcide/wodin-proxy:latest --ssl example.com
```

Then copy in the certificate as `/run/proxy/certificate.pem` and the key as `/run/proxy/key.pem` (the container will print these expected paths in its logs on startup)

```
docker cp ssl/certificate.pem proxy:/run/proxy/
docker cp ssl/key.pem proxy:/run/proxy/
```

Once the certificate files are found, then the proxy will start running.

See [`example/`](example) for a full example
