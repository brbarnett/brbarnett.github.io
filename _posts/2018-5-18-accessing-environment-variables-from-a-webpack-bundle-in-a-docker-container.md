---
layout: post
title: Accessing environment variables from a webpack bundle in a Docker container
tags: webpack docker
---

So I have this React application, bundled with webpack and containerized in an image based on an Apache httpd base image. I want the app to make synchronous HTTP requests, but I don't want the container at build time to be aware of where it's making that request - this should be a configuration detail rather than implementation.

My first instinct was to pass environment variables into the container at runtime. While that didn't end up being the wrong direction, it's not so simple because webpack only picks up environment variables at build time, not run time. As that bundle gets served, it has no awareness of the container's environment variables.

But you can fix that. It's not the sexiest solution by any means, but I found in my case that it was as simple as auto-generating a config.js file at container startup time, where I can output the relevant environment variables as a JSON object.

### Let's jump into code
Let's start by creating a `config.sh` script that gets copied into image at build:
```
#!/bin/bash
echo "window.appConfig = { API_URL: '${!API_URL}'} " >> config.js
nginx -g "daemon off;"
```

_**Warning**: if you wrote this script with Windows, you need to change the EOL characters to Unix format. Remember that these commands execute on Linux, and your Windows characters will throw cryptic errors._

The job of the config script is to create a JavaScript file that outputs a configuration JSON object to `window.appConfig` (this is configurable - I didn't want to use `window.config` becuase I was concerned about collisions). Since this overrides the default startup command, it also needs to run a command to launch nginx.

On top of copying the `config.sh` script into the image, you will also need to set permissions and execute in your `Dockerfile`:
```
RUN ["chmod", "+x", "./config.sh"]

CMD ["/usr/local/apache2/htdocs/config.sh"]
```

Finally, pass in the environment variable in `docker-compose.yaml` - I'm simplifying, but this will work with Docker Swarm, Kubernetes, etc. Wherever you can orchastrate containers with environment variables.
```
...
environment:
      - API_URL=http://localhost:8082
...
```

Now, it's as simple as referencing the auto-generated `config.js` from the `index.html` file. Your app can access environment variables from the `window.appConfig` object.
```
<!-- config.js has to come before your bundle -->
<script src="config.js"></script>  
<script src="dist/bundle.js"></script>
```

I'm curious to learn of other solutions to this problem - this seems oversimplified but it worked for what I needed.