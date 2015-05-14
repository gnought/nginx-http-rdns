# Nginx HTTP rDNS module with x_forwarded_for header support

## Summary

This module allows to make a reverse DNS (rDNS) lookup for incoming
connection and provides simple access control of incoming hostname
by allow/deny rules (similar to HttpAccessModule allow/deny
directives; regular expressions are supported). Module works with
the DNS server defined by the standard resolver directive.


## Example

    location / {
        resolver 8.8.8.8;
        
        rdns_proxy 10.132.17.146;
        rdns_proxy_recursive on;
        
        rdns_deny badone\.example\.com;

        if ($http_user_agent ~* FooAgent) {
            rdns on;
        }

        if ($rdns_hostname ~* (foo\.example\.com)) {
            set $myvar foo;
        }

        #...
    }

In the example above, nginx will make a reverse DNS request (through
the 8.8.8.8 DNS server) for each request having the "FooAgent"
user agent. Requests from badone.example.com will be forbidden.
The $rdns_hostname variable will have the rDNS request result or
"not found" (in case it's not found or any error occured) for any
requests made by FooAgent. For other user agents, $rdns_hostname
will have a special value "-".


## Directives

### rdns

* Syntax: rdns on | off | double
* Default: -
* Context: http, server, location, if-in-server, if-in-location
* Phase: rewrite
* Variables: rdns_hostname

Enables/disables rDNS lookups.

* on     - enable rDNS lookup in this context.
* double - enable double DNS lookup in this context. If the reverse
           lookup (rDNS request) succeeded, module performs a forward
           lookup (DNS request) for its result. If this forward
           lookup has failed or none of the forward lookup IP
           addresses have matched the original address,
           $rdns_hostname is set to "not found".
* off    - disable rDNS lookup in this context.

The $rdns_hostname variable may have:

* result of lookup;
* special value "not found" if not found or error occurred during
  request;
* special value "-" if lookup disabled.

After performing a lookup, module restarts request handling pipeline
to make new $rdns_hostname variable value visible to other directives.

Notice on server/location "if":

Internally, in server's or location's "if", module works through
rewrite module codes. When any enabling directive (rdns on|double) is
executed for the first time, it enables DNS lookup and makes a break
(to prevent executing further directives in this "if"). After the
lookup is done, directives in "if" using rewrite module codes are
executed for the second time, without any breaks. Disabling directive
(rdns off) makes no breaks.

Core module resolver should be defined to use this directive.


### rdns_proxy

* Syntax: rdns_proxy address | CIDR
* Default: -
* Context: http, server, location
* Phase: rewrite
* Variables: -

Defines trusted addresses. When a request comes from a trusted address, an address from the “X-Forwarded-For” request header field will be used instead.


### rdns_proxy_recursive

* Syntax: rdns_proxy_recursive on | off
* Default: -
* Context: http, server, location
* Phase: rewrite
* Variables: -

If recursive search is disabled then instead of the original client address that matches one of the trusted addresses, the last address sent in “X-Forwarded-For” will be used. If recursive search is enabled then instead of the original client address that matches one of the trusted addresses, the last non-trusted address sent in “X-Forwarded-For” will be used.


### rdns_allow

* Syntax: rdns_allow regex
* Default: -
* Context: http, server, location
* Phase: access
* Variables: -

Grants access for domain matched by regular expression.


### rdns_deny

* Syntax: rdns_deny regex
* Default: -
* Context: http, server, location
* Phase: access
* Variables: -

Forbids access for domain matched by regular expression.


## Notice on access lists

The rdns_allow and rdns_deny directives define a new access list for
the context in which they are used.

Access list inheritance in contexts works only if child context
doesn't define own rules.


## Warning on named locations

Making rDNS requests in named locations isn't supported and may
invoke a loop. For example:

    server {
        rdns on;

        location / {
            echo_exec @foo;
        }

        location @foo {
            #...
        }
    }

Being in a named location and restarting request handling pipeline,
nginx continue its request handling in usual (unnamed) location.
That's why this example will make a loop if you don't disable the
module in your named location. The correct config for this example
should be as follows:

    server {
        rdns on;

        location / {
            echo_exec @foo;
        }

        location @foo {
            rdns off;
            #...
        }
    }


## Authors

The original version of this module has been designed by 
Dmitry Stolyarov, written by Timofey Kirillov, CJSC Flant 
(flant.com).


## Links

* The source code on GitHub:
  https://github.com/uiltondutra/nginx-http-rdns
