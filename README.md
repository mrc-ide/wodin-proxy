# wodin-proxy

Proxy configuration for wodin. This can bring up [nginx](https://nginx.org/) in either http or https mode, pointing at a number of wodin instances. Generally we'll assume that these are running each on a different container, but they could also be running on the same machine but with a series of ports.

You need a directory, say `sites/` containing small snippets of configuration describing each site:

```
location /demo {
    return 301 $scheme://$host/demo/;
}
location  /demo/ {
    rewrite /demo/(.*) /$1  break;
    proxy_pass         http://wodin-demo:3000/;
    proxy_redirect     off;
    proxy_set_header   Host $host;
}
```

The dollar variables should not be changed, nginx fills these in correctly for us. The thing to change here is:

* `/demo`; the path that the site should be available at within your site
* `wodin-demo:3000/`; the location of the wodin instance (here assuming it's a wodin container with name `wodin-demo`)

You also want a directory, say `root/` containing files to be served for paths that do not match any sites (e.g., a generic index page, 404 and 500 error files).

Bringing things up in http mode is easy:

```
docker run -d --network wodin-nw --name proxy \
       -p 80:80 \
       -v $PWD/root:/wodin/root:ro \
       -v $PWD/sites:/wodin/sites:ro \
       mrcide/wodin-proxy:latest localhost
```

Here, we use the `:ro` modifier on the bind mounts to emphasise that these are readonly mounts; the proxy will not modify anything here. The final argument is the hostname that the site will serve on. For a http-only site for local testing, `localhost` is a good idea, but for nothing else. We need to expose only port 80 to the host.

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

First, bring up the container (with the first three changes from above made

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
