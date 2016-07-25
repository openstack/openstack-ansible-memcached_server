OpenStack-Ansible Memcached Server
##################################

Ansible role to install and configure Memcached

Default Variables
=================

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.

Required Variables
==================

None

Example Playbook
================

.. code-block:: yaml

    - name: Install memcached
      hosts: memcached
      user: root
      roles:
        - { role: "memcached_server" }

