.. _config_udp_listener_filters_dns_filter:

DNS Filter
==========

.. attention::

  DNS Filter is under active development and should be considered alpha and not production ready.

* :ref:`v3 API reference <envoy_v3_api_msg_extensions.filters.udp.dns_filter.v3alpha.DnsFilterConfig>`
* This filter should be configured with the name *envoy.filters.udp_listener.dns_filter*

Overview
--------

The DNS filter allows Envoy to resolve forward DNS queries as an authoritative server for all
configured domains. The filter's configuration specifies the names and addresses for which Envoy
will answer as well as the configuration needed to send queries externally for unknown domains.

The filter supports local and external DNS resolution. If a lookup for a name does not match a
statically configured domain, or a provisioned cluster name, Envoy can refer the query to an
external resolver for an answer. Users have the option of specifying the DNS servers that Envoy
will use for external resolution.

The filter supports :ref:`per-filter configuration
<envoy_v3_api_msg_extensions.filters.udp.dns_filter.v3alpha.DnsFilterConfig>`.
An Example configuration follows that illustrates how the filter can be used.

Example Configuration
---------------------

.. code-block:: yaml

  listener_filters:
    name: "envoy.filters.udp.dns_filter"
    typed_config:
      "@type": "type.googleapis.com/envoy.extensions.filters.udp.dns_filter.v3alpha.DnsFilterConfig"
      stat_prefix: "dns_filter_prefix"
      client_config:
        resolution_timeout: 5s
        upstream_resolvers:
        - "8.8.8.8"
        - "8.8.4.4"
      server_config:
        inline_dns_table:
          known_suffixes:
          - suffix: "domain1.com"
          - suffix: "domain2.com"
          - suffix: "domain3.com"
          virtual_domains:
           - name: "www.domain1.com"
             endpoint:
               address_list:
                 address:
                 - 10.0.0.1
                 - 10.0.0.2
            - name: "www.domain2.com"
              endpoint:
                address_list:
                  address:
                    - 2001:8a:c1::2800:7
            - name: "www.domain3.com"
              endpoint:
                address_list:
                  address:
                  - 10.0.3.1


In this example, Envoy is configured to respond to client queries for three domains. For any
other query, it will forward upstream to external resolvers. The filter will return an address
matching the input query type. If the query is for type A records and no A records are configured,
Envoy will return no addresses and set the response code appropriately. Conversely, if there are
matching records for the query type, each configured address is returned. This is also true for
AAAA records. Only A and AAAA records are supported. If the filter parses other queries for other
record types, the filter immediately responds indicating that the query is not supported.

To disable external resolution, one can omit the `client_config` section of the config. Envoy interprets
this configuration to mean that name resolution should be done only from the data appearing in the
`server_config` section. A query for a name not appearing in the DNS table will receive a "No Answer"
DNS response.

The filter can also consume its configuration from an external dns table. The same configuration
that appears in the static configuration can be stored in a Proto3-conformant JSON file and
referenced in the configuration using the :ref:`external_dns_table DataSource <envoy_api_msg_core.DataSource>`
directive:

Example External DnsTable Configuration
---------------------------------------

.. code-block:: yaml

    listener_filters:
      name: "envoy.filters.udp.dns_filter"
      typed_config:
        '@type': 'type.googleapis.com/envoy.extensions.filters.udp.dns_filter.v3alpha.DnsFilterConfig'
        stat_prefix: "my_prefix"
        server_config:
          external_dns_table:
            filename: "/home/ubuntu/configs/dns_table.json"

In the file, the table can be defined as follows:

DnsTable JSON Configuration
---------------------------

.. code-block:: text

  known_suffixes: [
    { suffix: "suffix1.com" },
    { suffix: "suffix2.com" }
  ],
  virtual_domains: [
    {
      name: "www.suffix1.com",
      endpoint: {
        address_list: {
          address: [ "10.0.0.1", "10.0.0.2" ]
        }
      }
    },
    {
      name: "www.suffix2.com",
      endpoint: {
        address_list: {
          address: [ "2001:8a:c1::2800:7" ]
        }
      }
    }
  ]


By utilizing this configuration, the DNS responses can be configured separately from the Envoy
configuration.
