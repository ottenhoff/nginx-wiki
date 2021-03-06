
.. meta::
   :description: The SR Cache module provides a transparent caching layer for arbitrary NGINX locations. The caching behavior is mostly compatible with RFC 2616.

SR Cache
========

Name
----
**ngx_srcache** - Transparent subrequest-based caching layout for arbitrary NGINX locations

.. note:: *This module is not distributed with the NGINX source.* See the `installation instructions <sr_cache.installation_>`_.



Status
------
This module is production ready.



Version
-------
This document describes srcache-nginx-module :github:`v0.29 <openresty/srcache-nginx-module/tags>` released on February 18, 2015.



Synopsis
--------
.. code-block:: nginx

  upstream my_memcached {
      server 10.62.136.7:11211;
      keepalive 10;
  }

  location = /memc {
      internal;

      memc_connect_timeout 100ms;
      memc_send_timeout 100ms;
      memc_read_timeout 100ms;
      memc_ignore_client_abort on;

      set $memc_key $query_string;
      set $memc_exptime 300;

      memc_pass my_memcached;
  }

  location /foo {
      set $key $uri$args;
      srcache_fetch GET /memc $key;
      srcache_store PUT /memc $key;
      srcache_store_statuses 200 301 302;

      # proxy_pass/fastcgi_pass/drizzle_pass/echo/etc...
      # or even static files on the disk
  }

.. code-block:: nginx

  location = /memc2 {
      internal;

      memc_connect_timeout 100ms;
      memc_send_timeout 100ms;
      memc_read_timeout 100ms;
      memc_ignore_client_abort on;

      set_unescape_uri $memc_key $arg_key;
      set $memc_exptime $arg_exptime;

      memc_pass unix:/tmp/memcached.sock;
  }

  location /bar {
      set_escape_uri $key $uri$args;
      srcache_fetch GET /memc2 key=$key;
      srcache_store PUT /memc2 key=$key&exptime=$srcache_expire;

      # proxy_pass/fastcgi_pass/drizzle_pass/echo/etc...
      # or even static files on the disk
  }

.. code-block:: nginx

  map $request_method $skip_fetch {
      default     0;
      POST        1;
      PUT         1;
  }

  server {
      listen 8080;

      location /api/ {
          set $key "$uri?$args";

          srcache_fetch GET /memc $key;
          srcache_store PUT /memc $key;

          srcache_methods GET PUT POST;
          srcache_fetch_skip $skip_fetch;

          # proxy_pass/drizzle_pass/content_by_lua/echo/...
      }
  }



Description
-----------
This module provides a transparent caching layer for arbitrary NGINX locations (like those use an upstream or even serve static disk files). The caching behavior is mostly compatible with `RFC 2616 <http://www.ietf.org/rfc/rfc2616.txt>`_.

Usually, :doc:`memc` is used together with this module to provide a concrete caching storage backend. But technically, any modules that provide a REST interface can be used as the fetching and storage subrequests used by this module.

For main requests, the `srcache_fetch`_ directive works at the end of the access phase, so the `standard access module <|HttpAccessModule|>`_'s `allow <|HttpAccessModule|#allow>`_ and `deny <|HttpAccessModule|#deny>`_ direcives run *before* ours, which is usually the desired behavior for security reasons.

The workflow of this module looks like below:

http://agentzh.org/misc/image/srcache-flowchart.png



Subrequest caching
^^^^^^^^^^^^^^^^^^
For *subrequests*, we explicitly **disallow** the use of this module because it's too difficult to get right. There used to be an implementation but it was buggy and I finally gave up fixing it and abandoned it.

However, if you're using :doc:`lua`, it's easy to do subrequest caching in Lua all by yourself. That is, first issue a subrequest to an :doc:`memc` location to do an explicit cache lookup, if cache hit, just use the cached data returned; otherwise, fall back to the true backend, and finally do a cache insertion to feed the data into the cache.

Using this module for main request caching and Lua for subrequest caching is the approach that we're taking in our business. This hybrid solution works great in production.



Distributed Memcached Caching
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Here is a simple example demonstrating a distributed memcached caching mechanism built atop this module. Suppose we do have three different memcacached nodes and we use simple modulo to hash our keys.

.. code-block:: nginx

  http {
      upstream moon {
          server 10.62.136.54:11211;
          server unix:/tmp/memcached.sock backup;
      }

      upstream earth {
          server 10.62.136.55:11211;
      }

      upstream sun {
          server 10.62.136.56:11211;
      }

      upstream_list universe moon earth sun;

      server {
          memc_connect_timeout 100ms;
          memc_send_timeout 100ms;
          memc_read_timeout 100ms;

          location = /memc {
              internal;

              set $memc_key $query_string;
              set_hashed_upstream $backend universe $memc_key;
              set $memc_exptime 3600; # in seconds
              memc_pass $backend;
          }

          location / {
              set $key $uri;
              srcache_fetch GET /memc $key;
              srcache_store PUT /memc $key;

              # proxy_pass/fastcgi_pass/content_by_lua/drizzle_pass/...
          }
      }
  }


Here's what is going on in the sample above:

#. We first define three upstreams, ``moon``, ``earth``, and ``sun``. These are our three memcached servers.
#. And then we group them together as an upstream list entity named ``universe`` with the ``upstream_list`` directive provided by :doc:`set_misc`.
#. After that, we define an internal location named ``/memc`` for talking to the memcached cluster.
#. In this ``/memc`` location, we first set the ``$memc_key`` variable with the query string (``$args``), and then use the ``set_hashed_upstream`` directive to hash our ``$memc_key`` over the upsteam list ``universe``, so as to obtain a concrete upstream name to be assigned to the variable ``$backend``.
#. We pass this ``$backend`` variable into the ``memc_pass`` directive. The ``$backend`` variable can hold a value among ``moon``, ``earth``, and ``sun``.
#. Also, we define the memcached caching expiration time to be 3600 seconds (i.e., an hour) by overriding the ``$memc_exptime`` variable.
#. In our main public location ``/``, we configure the ``$uri`` variable as our cache key, and then configure `srcache_fetch`_ for cache lookups and `srcache_store`_ for cache updates. We're using two subrequests to our ``/memc`` location defined earlier in these two directives.

One can use :doc:`lua`'s ``set_by_lua`` or ``rewrite_by_lua`` directives to inject custom Lua code to compute the ``$backend`` and/or ``$key`` variables in the sample above.

One thing that should be taken care of is that memcached does have restriction on key lengths, i.e., 250 bytes, so for keys that may be very long, one could use the ``set_md5`` directive or its friends to pre-hash the key to a fixed-length digest before assigning it to ``$memc_key`` in the ``/memc`` location or the like.

Further, one can utilize the `srcache_fetch_skip`_ and `srcache_store_skip`_ directives to control what to cache and what not on a per-request basis, and Lua can also be used here in a similar way. So the possibility is really unlimited.

To maximize speed, we often enable TCP (or Unix Domain Socket) connection pool for our memcached upstreams provided by `keepalive <http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive>`_, for example,

.. code-block:: nginx

  upstream moon {
      server 10.62.136.54:11211;
      server unix:/tmp/memcached.sock backup;
      keepalive 10;
  }


where we define a connection pool which holds up to 10 keep-alive connections (per NGINX worker process) for our ``moon`` upstream (cluster).



Caching with Redis
^^^^^^^^^^^^^^^^^^
One annoyance with Memcached backed caching is Memcached server's 1 MB value size limit. So it is often desired to use some more permissive backend storage services like Redis to serve as this module's backend.

Here is a working example by using Redis:

.. code-block:: nginx

  location /api {
      default_type text/css;

      set $key $uri;
      set_escape_uri $escaped_key $key;

      srcache_fetch GET /redis $key;
      srcache_store PUT /redis2 key=$escaped_key&exptime=120;

      # fastcgi_pass/proxy_pass/drizzle_pass/postgres_pass/echo/etc
  }

  location = /redis {
      internal;

      set_md5 $redis_key $args;
      redis_pass 127.0.0.1:6379;
  }

  location = /redis2 {
      internal;

      set_unescape_uri $exptime $arg_exptime;
      set_unescape_uri $key $arg_key;
      set_md5 $key;

      redis2_query set $key $echo_request_body;
      redis2_query expire $key $exptime;
      redis2_pass 127.0.0.1:6379;
  }


This example makes use of the ``$echo_request_body`` variable provided by :doc:`echo`. Note that you need the latest version of :doc:`echo`, ``v0.38rc2`` because earlier versions may not work reliably.

Also, you need both :doc:`redis` and :doc:`redis2`. The former is used in the `srcache_fetch`_ subrequest and the latter is used in the `srcache_store`_ subrequest.

The NGINX core also has a bug that could prevent :doc:`redis2`'s pipelining support from working properly in certain extreme conditions. And the following patch fixes this::

   http://mailman.nginx.org/pipermail/nginx-devel/2012-March/002040.html

Note that, however, if you are using the `ngx_openresty <http://openresty.org/>`_ 1.0.15.3 bundle or later, then you already have everything that you need here in the bundle.



Cache Key Preprocessing
^^^^^^^^^^^^^^^^^^^^^^^
It is often desired to preprocess the cache key to exclude random noises that may hurt the cache hit rate. For example, random session IDs in the URI arguments are usually desired to get removed.

Consider the following URI querystring::

    SID=BC3781C3-2E02-4A11-89CF-34E5CFE8B0EF&UID=44332&L=EN&M=1&H=1&UNC=0&SRC=LK&RT=62


we want to remove the ``SID`` and ``UID`` arguments from it. It is easy to achieve if you use :doc:`lua` at the same time:

.. code-block:: nginx

  location = /t {
      rewrite_by_lua '
          local args = ngx.req.get_uri_args()
          args.SID = nil
          args.UID = nil
          ngx.req.set_uri_args(args)
      ';

      echo $args;
  }


Here we use the ``echo`` directive from :doc:`echo` to dump out the final value of `$args <|HttpCoreModule|#$args>`_ in the end. You can replace it with your :doc:`sr_cache` configurations and upstream configurations instead for your case. Let's test this /t interface with curl:

.. code-block:: bash

  $ curl 'localhost:8081/t?RT=62&SID=BC3781C3-2E02-4A11-89CF-34E5CFE8B0EF&UID=44332&L=EN&M=1&H=1&UNC=0&SRC=LK'
  M=1&UNC=0&RT=62&H=1&L=EN&SRC=LK


It is worth mentioning that, if you want to retain the order of the URI arguments, then you can do string substitutions on the value of `$args <|HttpCoreModule|#$args>`_ directly, for example:

.. code-block:: nginx

  location = /t {
      rewrite_by_lua '
          local args = ngx.var.args
          newargs, n, err = ngx.re.gsub(args, [[\b[SU]ID=[^&]*&?]], "", "jo")
          if n and n > 0 then
              ngx.var.args = newargs
          end
      ';

      echo $args;
  }


Now test it with the original curl command again, we get exactly what we would expect::

  RT=62&L=EN&M=1&H=1&UNC=0&SRC=LK


But for caching purposes, it's good to normalize the URI argument order so that you can increase the cache hit rate. And the hash table entry order used by LuaJIT or Lua can be used to normalize the order as a nice side effect.



Directives
----------
srcache_fetch
^^^^^^^^^^^^^
:Syntax: *srcache_fetch <method> <uri> [args]...*
:Default: *none*
:Context: *http, server, location, location if*
:Phase: *post-access*

This directive registers an access phase handler that will issue an NGINX subrequest to lookup the cache.

When the subrequest returns status code other than ``200``, than a cache miss is signaled and the control flow will continue to the later phases including the content phase configured by |HttpProxyModule|, |HttpFastCGIModule|, and others. If the subrequest returns ``200 OK``, then a cache hit is signaled and this module will send the subrequest's response as the current main request's response to the client directly.

This directive will always run at the end of the access phase, such that |HttpAccessModule|'s `allow <|HttpAccessModule|#allow>`_ and `deny <|HttpAccessModule|#deny>`_ will always run *before* this.

You can use the `srcache_fetch_skip`_ directive to disable cache look-up selectively.



srcache_fetch_skip
^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_fetch_skip <flag>*
:Default: *0*
:Context: *http, server, location, location if*
:Phase: *post-access*

The ``<flag>`` argument supports NGINX variables. When this argument's value is not empty *and* not equal to ``0``, then the fetching process will be unconditionally skipped.

For example, to skip caching requests which have a cookie named ``foo`` with the value ``bar``, we can write

.. code-block:: nginx

  location / {
      set $key ...;
      set_by_lua $skip '
          if ngx.var.cookie_foo == "bar" then
              return 1
          end
          return 0
      ';

      srcache_fetch_skip $skip;
      srcache_store_skip $skip;

      srcache_fetch GET /memc $key;
      srcache_store GET /memc $key;

      # proxy_pass/fastcgi_pass/content_by_lua/...
  }


where :doc:`lua` is used to calculate the value of the ``$skip`` variable at the (earlier) rewrite phase. Similarly, the ``$key`` variable can be computed by Lua using the `set_by_lua <|HttpLuaModule|#set_by_lua>`_ or `rewrite_by_lua <|HttpLuaModule|#rewrite_by_lua>`_ directive too.

The standard `map <|HttpMapModule|#map>`_ directive can also be used to compute the value of the ``$skip`` variable used in the sample above:

.. code-block:: nginx

  map $cookie_foo $skip {
      default     0;
      bar         1;
  }


but your `map <|HttpMapModule|#map>`_ statement should be put into the ``http`` config block in your ``nginx.conf`` file though.



srcache_store
^^^^^^^^^^^^^
:Syntax: *srcache_store <method> <uri> [args]...*
:Default: *none*
:Context: *http, server, location, location if*
:Phase: *output-filter*

This directive registers an output filter handler that will issue an NGINX subrequest to save the response of the current main request into a cache backend. The status code of the subrequest will be ignored.

You can use the `srcache_store_skip`_ and `srcache_store_max_size`_ directives to disable caching for certain requests in case of a cache miss.

Since the ``v0.12rc7`` release, both the response status line, response headers, and response bodies will be put into the cache. By default, the following special response headers will not be cached:

* Connection
* Keep-Alive
* Proxy-Authenticate
* Proxy-Authorization
* TE
* Trailers
* Transfer-Encoding
* Upgrade
* Set-Cookie

You can use the `srcache_store_pass_header`_ and/or `srcache_store_hide_header`_ directives to control what headers to cache and what not.

The original response's data chunks get emitted as soon as they arrive. ``srcache_store`` just copies and collects the data in an output filter without postponing them from being sent downstream.

But please note that even though all the response data will be sent immediately, the current NGINX request lifetime will not finish until the srcache_store subrequest completes. That means a delay in closing the TCP connection on the server side (when HTTP keepalive is disabled, but proper HTTP clients should close the connection actively on the client side, which adds no extra delay or other issues at all) or serving the next request sent on the same TCP connection (when HTTP keepalive is in action).



srcache_store_max_size
^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_max_size <size>*
:Default: *0*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

When the response body length is exceeding this size, this module will not try to store the response body into the cache using the subrequest template that is specified in `srcache_store`_.

This is particular useful when using cache storage backend that does have a hard upper limit on the input data. For example, for Memcached server, the limit is usually ``1 MB``.

When ``0`` is specified (the default value), there's no limit check at all.



srcache_store_skip
^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_skip <flag>*
:Default: *0*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

The ``<flag>`` argument supports NGINX variables. When this argument's value is not empty *and* not equal to ``0``, then the storing process will be unconditionally skipped.

Starting from the ``v0.25`` release, the ``<flag>`` expression (possibly containing NGINX variables) can be evaluated up to twice: the first time is right after the response header is being sent and when the ``<flag>`` expression is not evaluated to true values it will be evaluated again right after the end of the response body data stream is seen. Before ``v0.25``, only the first time evaluation is performed.

Here's an example using Lua to set $nocache to avoid storing URIs that contain the string "/tmp":

.. code-block:: nginx

  set_by_lua $nocache '
      if string.match(ngx.var.uri, "/tmp") then
          return 1
      end
      return 0';

  srcache_store_skip $nocache;




srcache_store_statuses
^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_statuses <status1> <status2>...*
:Default: *200 301 302*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

This directive controls what responses to store to the cache according to their status code.

By default, only ``200``, ``301``, and ``302`` responses will be stored to cache and any other responses will skip `srcache_store`_.

You can specify arbitrary positive numbers for the response status code that you'd like to cache, even including error code like ``404`` and ``503``. For example:

.. code-block:: nginx

  srcache_store_statuses 200 201 301 302 404 503;


At least one argument should be given to this directive.

This directive was first introduced in the ``v0.13rc2`` release.



srcache_store_ranges
^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_ranges [ on | off ]*
:Default: *off*
:Context: *http, server, location, location if*
:Phase: *output-body-filter*

When this directive is turned on, `srcache_store`_ will also store 206 Partial Content responses generated by the standard ``ngx_http_range_filter_module``. If you turn this directive on, you MUST add ``$http_range`` to your cache keys. For example,

.. code-block:: nginx

  location / {
      set $key "$uri$args$http_range";
      srcache_fetch GET /memc $key;
      srcache_store PUT /memc $key;
  }


This directive was first introduced in the ``v0.27`` release.



srcache_header_buffer_size
^^^^^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_header_buffer_size <size>*
:Default: *4k/8k*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

This directive controles the header buffer when serializing response headers for `srcache_store`_. The default size is the page size, usually ``4k`` or ``8k`` depending on specific platforms.

Note that the buffer is not used to hold all the response headers, but just each individual header. So the buffer is merely needed to be big enough to hold the longest response header.

This directive was first introduced in the ``v0.12rc7`` release.



srcache_store_hide_header
^^^^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_hide_header <header>*
:Default: *no*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

By default, this module caches all the response headers except the following ones:

* Connection
* Keep-Alive
* Proxy-Authenticate
* Proxy-Authorization
* TE
* Trailers
* Transfer-Encoding
* Upgrade
* Set-Cookie

You can hide even more response headers from `srcache_store`_ by listing their names (case-insensitive) by means of this directive. For examples,

.. code-block:: nginx

  srcache_store_hide_header X-Foo;
  srcache_store_hide_header Last-Modified;


Multiple occurrences of this directive are allowed in a single location.

This directive was first introduced in the ``v0.12rc7`` release.

See also `srcache_store_pass_header`_.



srcache_store_pass_header
^^^^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_pass_header <header>*
:Default: *no*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

By default, this module caches all the response headers except the following ones:

* Connection
* Keep-Alive
* Proxy-Authenticate
* Proxy-Authorization
* TE
* Trailers
* Transfer-Encoding
* Upgrade
* Set-Cookie

You can force `srcache_store`_ to store one or more of these response headers from `srcache_store`_ by listing their names (case-insensitive) by means of this directive. For examples,

.. code-block:: nginx

  srcache_store_pass_header Set-Cookie;
  srcache_store_pass_header Proxy-Autenticate;


Multiple occurrences of this directive are allowed in a single location.

This directive was first introduced in the ``v0.12rc7`` release.

See also `srcache_store_hide_header`_.



srcache_methods
^^^^^^^^^^^^^^^
:Syntax: *srcache_methods <method>...*
:Default: *GET HEAD*
:Context: *http, server, location*
:Phase: *post-access, output-header-filter*

This directive specifies HTTP request methods that are considered by either `srcache_fetch`_ or `srcache_store`_. HTTP request methods not listed will be skipped completely from the cache.

The following HTTP methods are allowed: ``GET``, ``HEAD``, ``POST``, ``PUT``, and ``DELETE``. The ``GET`` and ``HEAD`` methods are always implicitly included in the list regardless of their presence in this directive.

Note that since the ``v0.17`` release ``HEAD`` requests are always skipped by `srcache_store`_ because their responses never carry a response body.

This directive was first introduced in the ``v0.12rc7`` release.



srcache_ignore_content_encoding
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_ignore_content_encoding [ on | off ]*
:Default: *off*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

When this directive is turned ``off`` (which is the default), non-empty ``Content-Encoding`` response header will cause `srcache_store`_ skip storing the whole response into the cache and issue a warning into NGINX's ``error.log`` file like this:

.. code-block:: text

  [warn] 12500#0: *1 srcache_store skipped due to response header "Content-Encoding: gzip"
              (maybe you forgot to disable compression on the backend?)


Turning on this directive will ignore the ``Content-Encoding`` response header and store the response as usual (and also without warning).

It's recommended to always disable gzip/deflate compression on your backend server by specifying the following line in your ``nginx.conf`` file:

.. code-block:: nginx

  proxy_set_header  Accept-Encoding  "";


This directive was first introduced in the ``v0.12rc7`` release.



srcache_request_cache_control
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_request_cache_control [ on | off ]*
:Default: *off*
:Context: *http, server, location*
:Phase: *post-access, output-header-filter*

When this directive is turned ``on``, the request headers ``Cache-Control`` and ``Pragma`` will be honored by this module in the following ways:

#. `srcache_fetch`_, i.e., the cache lookup operation, will be skipped when request headers ``Cache-Control: no-cache`` and/or ``Pragma: no-cache`` are present.
#. `srcache_store`_, i.e., the cache store operation, will be skipped when the request header ``Cache-Control: no-store`` is specified.

Turning off this directive will disable this functionality and is considered safer for busy sites mainly relying on cache for speed.

This directive was first introduced in the ``v0.12rc7`` release.

See also `srcache_response_cache_control`_.



srcache_response_cache_control
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_response_cache_control [ on | off ]*
:Default: *on*
:Context: *http, server, location*
:Phase: *output-header-filter*

When this directive is turned ``on``, the response headers ``Cache-Control`` and ``Expires`` will be honored by this module in the following ways:

* ``Cache-Control: private`` skips `srcache_store`_,
* ``Cache-Control: no-store`` skips `srcache_store`_,
* ``Cache-Control: no-cache`` skips `srcache_store`_,
* ``Cache-Control: max-age=0`` skips `srcache_store`_,
* ``Expires: <date-no-more-recently-than-now>`` skips `srcache_store`_.

This directive takes priority over the `srcache_store_no_store`_, `srcache_store_no_cache`_, and `srcache_store_private`_ directives.

This directive was first introduced in the ``v0.12rc7`` release.

See also `srcache_request_cache_control`_.



srcache_store_no_store
^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_no_store [ on | off ]*
:Default: *off*
:Context: *http, server, location*
:Phase: *output-header-filter*

Turning this directive on will force responses with the header ``Cache-Control: no-store`` to be stored into the cache when `srcache_response_cache_control`_ is turned ``on`` *and* other conditions are met. Default to ``off``.

This directive was first introduced in the ``v0.12rc7`` release.



srcache_store_no_cache
^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_no_cache [ on | off ]*
:Default: *off*
:Context: *http, server, location*
:Phase: *output-header-filter*

Turning this directive on will force responses with the header ``Cache-Control: no-cache`` to be stored into the cache when `srcache_response_cache_control`_ is turned ``on`` *and* other conditions are met. Default to ``off``.

This directive was first introduced in the ``v0.12rc7`` release.



srcache_store_private
^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_store_private [ on | off ]*
:Default: *off*
:Context: *http, server, location*
:Phase: *output-header-filter*

Turning this directive on will force responses with the header ``Cache-Control: private`` to be stored into the cache when `srcache_response_cache_control`_ is turned ``on`` *and* other conditions are met. Default to ``off``.

This directive was first introduced in the ``v0.12rc7`` release.



srcache_default_expire
^^^^^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_default_expire <time>*
:Default: *60s*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

This directive controls the default expiration time period that is allowed for the `$srcache_expire`_ variable value when neither ``Cache-Control: max-age=N`` nor ``Expires`` are specified in the response headers.

The ``<time>`` argument values are in seconds by default. But it's wise to always explicitly specify the time unit to avoid confusion. Time units supported are "s"(seconds), "ms"(milliseconds), "y"(years), "M"(months), "w"(weeks), "d"(days), "h"(hours), and "m"(minutes). For example,

.. code-block:: nginx

  srcache_default_expire 30m; # 30 minutes


This time must be less than 597 hours.

This directive was first introduced in the ``v0.12rc7`` release.



srcache_max_expire
^^^^^^^^^^^^^^^^^^
:Syntax: *srcache_max_expire <time>*
:Default: *0*
:Context: *http, server, location, location if*
:Phase: *output-header-filter*

This directive controls the maximal expiration time period that is allowed for the `$srcache_expire`_ variable value. This setting takes priority over other calculating methods.

The ``<time>`` argument values are in seconds by default. But it's wise to always explicitly specify the time unit to avoid confusion. Time units supported are "s"(seconds), "ms"(milliseconds), "y"(years), "M"(months), "w"(weeks), "d"(days), "h"(hours), and "m"(minutes). For example,

.. code-block:: nginx

  srcache_max_expire 2h;  # 2 hours


This time must be less than 597 hours.

When ``0`` is specified, which is the default setting, then there will be *no* limit at all.

This directive was first introduced in the ``v0.12rc7`` release.



Variables
---------

$srcache_expire
^^^^^^^^^^^^^^^
This NGINX integer value represents the recommended expiration time period (in seconds) for the current response being stored into the cache. The algorithm of computing the value is as follows:

#. When the response header ``Cache-Control: max-age=N`` is specified, then ``N`` will be used as the expiration time,
#. otherwise if the response header ``Expires`` is specified, then the expiration time will be obtained by subtracting the current time stamp from the time specified in the ``Expires`` header,
#. when neither ``Cache-Control: max-age=N`` nor ``Expires`` headers are specified, use the value specified in the `srcache_default_expire`_ directive.

The final value of this variable will be the value specified by the `srcache_max_expire`_ directive if the value obtained in the algorithm above exceeds the maximal value (if any).

You don't have to use this variable for the expiration time.

This variable was first introduced in the ``v0.12rc7`` release.



$srcache_fetch_status
^^^^^^^^^^^^^^^^^^^^^
This NGINX variable is evaluated to the status of the "fetch" phase for the caching system. Three values are possible, ``HIT``, ``MISS``, and ``BYPASS``.

When the "fetch" subrequest returns status code other than ``200`` or its response data is not well-formed, then this variable is evaluated to the value ``MISS``.

The value of this variable is only meaningful after the ``access`` request processing phase, or ``BYPASS`` is always given.

This variable was first introduced in the ``v0.14`` release.



$srcache_store_status
^^^^^^^^^^^^^^^^^^^^^
This NGINX variable gives the current caching status for the "store" phase. Two possible values, ``STORE`` and ``BYPASS`` can be obtained.

Because the responses for the "store" subrequest are always discarded, so the value of this variable will always be ``STORE`` as long as the "store" subrequest is actually issued.

The value of this variable is only meaningful at least when the request headers of the current (main) request are being sent. The final result can only be obtained after all the response body has been sent if the ``Content-Length`` response header is not specified for the main request.

This variable was first introduced in the ``v0.14`` release.



Known Issues
------------
* On certain systems, enabling aio and/or sendfile may stop `srcache_store`_ from working. You can disable them in the locations configured by `srcache_store`_.
* The `srcache_store`_ directive can not be used to capture the responses generated by :doc:`echo`'s subrequest directivees like ``echo_subrequest_async`` and ``echo_location``. You are recommended to use *ngx_lua* to initiate and capture subrequests, which should work with `srcache_store`_.



Caveats
-------
* It is recommended to disable your backend server's gzip compression and use NGINX's |HttpGzipModule| to do the job. In case of |HttpProxyModule|, you can use the following configure setting to disable backend gzip compression:

  .. code-block:: nginx

    proxy_set_header  Accept-Encoding  "";


* Do *not* use |HttpRewriteModule|'s `if <|HttpRewriteModule|#if>`_ directive in the same location as this module's, because "`if <|HttpRewriteModule|#if>`_ is evil". Instead, use |HttpMapModule| or :doc:`lua` combined with this module's `srcache_store_skip`_ and/or `srcache_fetch_skip`_ directives. For example:

  .. code-block:: nginx

    map $request_method $skip_fetch {
        default     0;
        POST        1;
        PUT         1;
    }

    server {
        listen 8080;

        location /api/ {
            set $key "$uri?$args";

            srcache_fetch GET /memc $key;
            srcache_store PUT /memc $key;

            srcache_methods GET PUT POST;
            srcache_fetch_skip $skip_fetch;

            # proxy_pass/drizzle_pass/content_by_lua/echo/...
        }
    }



Trouble Shooting
----------------
To debug issues, you should always check your NGINX ``error.log`` file first. If no error messages are printed, you need to enable the NGINX debugging logs to get more details, as explained in `debugging log <http://nginx.org/en/docs/debugging_log.html>`_.

Several common pitfalls for beginners:

* The original response carries a ``Cache-Control`` header that explicitly disables caching and you do not configure directives like `srcache_response_cache_control`_.
* The original response is already gzip compressed, which is not cached by default (see `srcache_ignore_content_encoding`_).



.. _sr_cache.installation:

Installation
------------
It is recommended to install this module as well as the NGINX core and many other goodies via the `ngx_openresty bundle <http://openresty.org>`__. It is the easiest way and most safe way to set things up. See OpenResty's `installation instructions <http://openresty.org/#Installation>`_ for details.

Alternatively, you can build NGINX with this module all by yourself:

* Grab the NGINX source code from `nginx.org <http://nginx.org>`_, for example, the version 1.7.10 (see [[#Compatibility|NGINX Compatibility]]),
* and then apply the patch to your NGINX source tree that fixes an important bug in the mainline NGINX core: https://raw.githubusercontent.com/openresty/ngx_openresty/master/patches/nginx-1.4.3-upstream_truncation.patch (you do NOT need this patch if you are using NGINX 1.5.3 and later versions.)
* after that, download the latest version of the release tarball of this module from srcache-nginx-module :github:`file list <openresty/srcache-nginx-module/tags>`
* and finally build the NGINX source with this module
  
  .. code-block:: nginx

    wget 'http://nginx.org/download/nginx-1.7.10.tar.gz'
    tar -xzvf nginx-1.7.10.tar.gz
    cd nginx-1.7.10/

    # Here we assume you would install you nginx under /opt/nginx/.
    ./configure --prefix=/opt/nginx \
         --add-module=/path/to/srcache-nginx-module

    make -j2
    make install



Compatibility
-------------
The following versions of NGINX should work with this module:

* **1.7.x** (last tested: 1.7.10)
* **1.5.x** (last tested: 1.5.12)
* **1.4.x** (last tested: 1.4.4)
* **1.3.x** (last tested: 1.3.7)
* **1.2.x** (last tested: 1.2.9)
* **1.1.x** (last tested: 1.1.5)
* **1.0.x** (last tested: 1.0.11)
* **0.9.x** (last tested: 0.9.4)
* **0.8.x** >= 0.8.54 (last tested: 0.8.54)

Earlier versions of NGINX like 0.7.x, 0.6.x and 0.5.x will *not* work.

If you find that any particular version of NGINX above 0.7.44 does not work with this module, please consider reporting a bug.



.. _sr_cache.community:



Community
---------

English Mailing List
^^^^^^^^^^^^^^^^^^^^
The `openresty-en <https://groups.google.com/forum/#!forum/openresty-en>`_ mailing list is for English speakers.


Chinese Mailing List
^^^^^^^^^^^^^^^^^^^^
The `openresty <https://groups.google.com/forum/#!forum/openresty>`_ mailing list is for Chinese speakers.



Bugs and Patches
----------------
Please submit bug reports, wishlists, or patches by

#. creating a ticket on the :github:`GitHub Issue Tracker <openresty/srcache-nginx-module/issues>`
#. or posting to the `OpenResty community <sr_cache.community_>`_.



Source Repository
-----------------
Available on github at :github:`openresty/srcache-nginx-module <openresty/srcache-nginx-module>`



Test Suite
----------
This module comes with a Perl-driven test suite. The :github:`test cases <openresty/srcache-nginx-module/tree/master/t>` are :github:`declarative <openresty/srcache-nginx-module/blob/master/t/main-req.t>` too. Thanks to the `Test::Nginx <http://search.cpan.org/perldoc?Test::Base>` module in the Perl world.

To run it on your side:

.. code-block:: bash

  $ PATH=/path/to/your/nginx-with-srcache-module:$PATH prove -r t


You need to terminate any NGINX processes before running the test suite if you have changed the NGINX server binary.

Because a single NGINX server (by default, ``localhost:1984``) is used across all the test scripts (``.t`` files), it's meaningless to run the test suite in parallel by specifying ``-jN`` when invoking the ``prove`` utility.

Some parts of the test suite requires modules |HttpRewriteModule|, :doc:`echo`, :github:`HttpRdsJsonModule <openresty/rds-json-nginx-module>`, and :doc:`drizzle` to be enabled as well when building NGINX.



TODO
----
* add gzip compression and decompression support.
* add new NGINX variable ``$srcache_key`` and new directives ``srcache_key_ignore_args``, ``srcache_key_filter_args``, and ``srcache_key_sort_args``.



Getting involved
----------------
You'll be very welcomed to submit patches to the author or just ask for a commit bit to the source repository on GitHub.



Author
------
Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.



Copyright & License
-------------------
Copyright (c) 2010-2015, Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

This module is licensed under the terms of the BSD license.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.


.. seealso::

  * :doc:`memc`
  * :doc:`lua`
  * :doc:`set_misc`
  * The `ngx_openresty bundle <http://openresty.org>`__

