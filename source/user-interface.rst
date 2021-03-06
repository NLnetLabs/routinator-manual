User Interface
==============

Routinator offers a web based user interface. In order to start the HTTP
server at 192.0.2.13 and 2001:0DB8::13 on port 8323, run this command:

.. code-block:: text

   routinator server --http 192.0.2.13:8323 --http [2001:0DB8::13]:8323

The application will stay attached to your terminal unless you provide the
:option:`--detach` option. After the first validation run has completed, the
user interface from the example above will be available at 
http://192.0.2.13:8323/.

Prefix Check
------------

In addition to displaying detailed statistics from the last validation run
Routinator has performed, as well as HTTP and RTR connection metrics, the most
prominent functionality is the prefix check.

.. figure:: img/routinator-ui-prefix-check.png
    :align: center
    :width: 100%
    :alt: Routinator user interface

    The Routinator Prefix Check

By default, you only need to provide an IP address or prefix. When
clicking :guilabel:`Validate`, Routinator will look up from which Autonomous
System the closest matching prefix is announced in BGP and perform RPKI
validation. Alternatively, you can manually provide an ASN. 

The returned RPKI validity state will be *Valid*, *Invalid* or
*NotFound* and is based on the current set of Validated ROA Payloads (VRPs) in
the cache. Routinator will provide an overview of all VRPs that led to the
result, along with the reason for the outcome.

Routinator doesn't just retrieve the ASN for a specific prefix, but it also
fetches related information. In addition to validating the longest matching
prefix (or exact match if this is what you selected), details can be provided on
less specific and more specific announcements seen in BGP, as well as other
resources allocated to the same organisation.

.. figure:: img/routinator-ui-prefix-check-related.png
    :align: center
    :width: 100%
    :alt: Routinator user interface

    Prefixes related to your query

Routinator does not perform the BGP and allocation lookups itself, but relies on
the open source `roto-api <https://github.com/NLnetLabs/roto-api>`_ service,
developed and hosted by NLnet Labs at `bgp-api.net 
<https://rest.bgp-api.net/api/v1/>`_. The service uses these data sources:

  - BGP information based on `RISWhois <https://www.ris.ripe.net/dumps/>`_
    data, which is part of the RIPE NCC's `Routing Information Service 
    <https://ris.ripe.net/>`_ (RIS). This data set is currently updated every 8
    hours.
  - Resource allocations retrieved from `statistics 
    <https://www.nro.net/about/rirs/statistics/>`_ hosted by the five Regional 
    Internet Registries. These are updated daily.

Reverse Proxy with NGINX
------------------------

Routinator's built-in HTTP server is intended to run on your internal network
and doesn't offer HTTPS natively. If this is a requirement, you can for example
run Routinator behind an `NGINX <https://www.nginx.com>`_  reverse proxy. 

For convenience, all the files and folders for the user interface are hosted
under the ``/ui`` path. This allows you to just expose the user interface and
not any of the other paths, such as those serving the various VRP output
formats.

To only make the user interface available through a reverse proxy, this is what
your NGINX configuration needs at a minimum when running it on the same server
as Routinator runs on, using port 8323:

.. code-block:: text

    location = / {
      proxy_pass http://127.0.0.1:8323/;
    }
    location /ui {
      proxy_pass http://127.0.0.1:8323/ui;
    }
    location /api {
      proxy_pass http://127.0.0.1:8323/api;
    } 