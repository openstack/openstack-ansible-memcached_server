====================================
Alternative Memcached configurations
====================================

By default Memcached servers are deployed on each controller host as a part of
`shared-infra_containers` group. Drivers, like `oslo_cache.memcache_pool <https://github.com/openstack/oslo.cache/blob/master/oslo_cache/backends/memcache_pool.py>`_
support marking memcache backends as dead, however not all services allow you
to select the driver which will be used for interaction with Memcached.
In the meanwhile, you may face services API response delays or even
unresponsive APIs while one of the memcached backends is down.

This is why you may want to use HAProxy for handling access and to check for
backend aliveness or use always "local" to the service memcached server.

Configuring Memcached through HAProxy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Setting haproxy in front of the Memcached servers and relying on it for
checking aliveness of the backends gives more reliable failover and minimize
delays in case of backend failure. We need to define the following in your
``user_variables.yml``:

.. code-block:: yaml

   haproxy_memcached_allowlist_networks: "{{ haproxy_allowlist_networks }}"
   memcached_servers: "{{ internal_lb_vip_address ~ ':' ~ memcached_port }}"
   haproxy_extra_services:
     - service:
         haproxy_service_name: memcached
         haproxy_backend_nodes: "{{ groups['memcached'] | default([]) }}"
         haproxy_bind: "{{ [internal_lb_vip_address] }}"
         haproxy_port: 11211
         haproxy_balance_type: tcp
         haproxy_balance_alg: source
         haproxy_backend_ssl: False
         haproxy_backend_options:
           - tcp-check
         haproxy_allowlist_networks: "{{ haproxy_memcached_allowlist_networks }}"

After setting this you will need to update haproxy and all services
configuration to use new memcached backend:

.. code-block:: shell-session

  # openstack-ansible playbooks/haproxy-install.yml
  # openstack-ansible playbooks/setup-openstack.yml


Using only "local" Memcached
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The idea behind this, is to configure services to use Memcached, that will
reside only on the local control plane. Here "local" means not only having
memcahed inside the container with the service itslef, but also having
memcached inside a separate container on the same controller as the service
well.

This will reduce latency and improve stability, since service and memcached
instance will be running on the same control plane just on different containers
that are connected through the same L2 bridge.

Among the cons of this approach is that there won't be any failover available
in case of the memcached container crush, so caching on this controller won't
work. For pros it's only 1 controller that will be affected and not
all of them in the case of remote memcached unavailability. Also, most the
common scenario of API response delays is when whole controller goes down,
since connection drop causes memcache_pool to wait for connection timeout
rather then connection is rejected when memcached service goes down and
memcache_pool instantly switches to another backend.

.. note::

  In case some service won't have memcached server "locally"
  to the service that is running, behaviour will fallback to using
  all available memcached servers and memcached client will decide
  which one to use.

In order to always use "local" memcached you need to define the following
in the ``user_variables.yml`` file:

.. code-block:: yaml

  memcached_servers: |-
      {% set service_controller_group = group_names | select('regex', '.*-host_containers') | first | default('memcached') %}
      {{
        groups['memcached'] | intersect(groups[service_controller_group])
          | map('extract', hostvars, 'management_address')
          | map('regex_replace', '(.+)', '\1:' ~ memcached_port)
          | list | join(',')
      }}

After setting that you need to update all services configuration
to use new memcached backend:

.. code-block:: shell-session

  # openstack-ansible playbooks/setup-openstack.yml
