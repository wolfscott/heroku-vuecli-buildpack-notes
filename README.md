# VueJS CLI static file deployment to Heroku 

> 01-Oct-22 Wolf Scott  (www.ioihan.com)

## After the heroku-buildpack-static deprecation

The only constant in life is change the saying goes. Heroku decided to deprecate the **heroku-buildpack-static** build package and while they may have given a warning on this action, I sure didn't see it. It followed right on the heels of their mortarium on publishing code.

The replacement to this build pack was so not an obvious thing for me and took me some digging to find these pieces. If you know of a better way, please do share. I record these settings to hopefully save someone some time AND of course, I pen it here for my future-self as I have several more apps to swap over.

There are several steps to make this work. I am not a big command-line, script, fan when working with Heroku which evidently is frowned upon but these steps won't require any use of the Heroku cli.

I get frustrated quickly with these tutorials that say things like "how to publish..." and such titles and spend 95% of the tutorial showing how to create an app. But, to quote Richard, we digest.

## What we want to accomplish:

1. Swap out the build pack configuration
1. Drop in a few configuration files
1. Push your code

NOTE: You can find the code mentioned here in my repo:  

<https://github.com/wolfscott/heroku-vuecli-buildpack-notes>

> *NOTE: I am assuming that your Dyno is connected to GitHub*   
> 
### Step 1

Let’s start with the build pack.  You should remove the “Heroku-buildpack-static” build pack and add the “https://buildpack-registry.s3.amazonaws.com/buildpacks/heroku-community/nginx.tgz “ which then will look like this.  If you are unsure how to do this, [x] the old one, and select Add Buildpack button, and paste in the new item above.

![](docs/Aspose.Words.bf36cf5d-c41a-47ee-9fba-9fd0907918a0.001.png)

### Step 2

The second part of this involves adding two new files.  If you already have these files then I think the task would be to compare the settings carefully as I’ve tried the recommended “defaults” and they did not work for me unmodified.  In fairness, I’m not sure they were supposed to work out of the box, but I was spoiled with the old build pack – it just worked.  

So, the two files are:  (I’ll discuss what goes into them in a moment)

- \Procfile
- \config\nginx.conf.erb

I was thrown by the extension of .erb, but turns out that yep, that extension -IS- required.  The Procfile has a cap first letter and NO extension. The other is all lower, just saying.   There was also some mention of a unicorn file which totally messed me up but that file is not required, as you see the only file in the config folder. (note that that this config is in the root folder and NOT in the \src folder as would be the case for app-level config stuff) 

The pathnames are based on your apps root and in fact for my VueJS cli app, these are the actual pathnames, as shown for the config example. You should see the package.json and dist folders so that should clarify the “where”. 

![](docs/Aspose.Words.bf36cf5d-c41a-47ee-9fba-9fd0907918a0.002.png)

![](docs/Aspose.Words.bf36cf5d-c41a-47ee-9fba-9fd0907918a0.003.png)

**nginx.conf.erb** contents (this handles the nginx related config AND the url rewrite required when refreshing a VueJS SPA application to send all relative paths back to index.html)   I’ve included the config text here and you should be able to quickly find the different parts of interest. (or refer to the repo at the top of this document)

``` rb
# 01-Oct-22 :: Wolf Scott ::  wscott@ioihan.com
# -------------------------------------------------------------------------------
# Old Buildpack: https: //github.com/heroku/heroku-buildpack-nginx
# New Docs     : https: //github.com/wolfscott/heroku-vuecli-buildpack-notes
# -------------------------------------------------------------------------------
daemon off;
# Heroku dynos have at least 4 cores.
worker_processes <%= ENV['NGINX_WORKERS'] || 4 %>;

events {
	use epoll;
	accept_mutex on;
	worker_connections <%= ENV['NGINX_WORKER_CONNECTIONS'] || 1024 %>;
}

http {
	gzip on;
	gzip_comp_level 2;
	gzip_min_length 512;
	gzip_proxied any; # Heroku router sends Via header

	server_tokens off;

	log_format l2met 'measure#nginx.service=$request_time request_id=$http_x_request_id';
	access_log <%= ENV['NGINX_ACCESS_LOG_PATH'] || 'logs/nginx/access.log' %> l2met;
	error_log <%= ENV['NGINX_ERROR_LOG_PATH'] || 'logs/nginx/error.log' %>;


	include mime.types;
	default_type application/octet-stream; 
	sendfile on; 

	# Must read the body in 5 seconds.
	client_body_timeout <%= ENV['NGINX_CLIENT_BODY_TIMEOUT'] || 5 %>;

	server {
		listen <%= ENV["PORT"] %>;
		server_name _;
		keepalive_timeout 5;
		client_max_body_size <%= ENV['NGINX_CLIENT_MAX_BODY_SIZE'] || 1 %>M;

		# location / {
		# 	# force SSL/redirect http -> https
		# 	if ($http_x_forwarded_proto != "https") {
		# 	  return 301 https://$host$request_uri;
		# 	}

    #   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		# 	proxy_set_header Host $http_host;
		# 	proxy_redirect off;
		# 	proxy_pass http://127.0.0.0/;
      
    # }
    
    location / {

			if ($http_x_forwarded_proto != "https") {
			  return 301 https://$host$request_uri;
			}

      alias /app/dist/;
      try_files $uri /index.html;
    }

		root /app/dist; # path to your app
	}
}


```


**Procfile** contents:  (rather simple)

``` text
web: bin/start-nginx-solo
```

And for completeness sake, I have a static.json file that was required in the previous build process, but I do not think this is required for the new method.  I am “guessing” that perhaps this is not required since this information defined in the nginx.conf.erb file, but I include it here for reference.

**static.json** file (not required, just for reference)
``` json
{
  "root": "dist",
  "clean_urls": true,
  "https_only": true,
  "routes": {
    "/**": "index.html"
  }
}
```

### Step 3

Once you have these files in place and ready, you should be good to push by whatever method you normally use to deply your app.

#### Some notes

- The VueJS CLI application builds to a static site.  Thus, no dev-dependencice should be flagged as "missing" when launching the app.

- Some have mentioned that after deployment that the @vue/cli-service package is not found. I've seen many recommendations, but remember that this package is a development-environment (dev dependency) so if this is being required, you are not serving your new app from the /dist folder, which is a pre-built static resource.



