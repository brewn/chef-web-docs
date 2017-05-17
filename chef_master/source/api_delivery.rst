=====================================================
Chef Automate API
=====================================================
`[edit on GitHub] <https://github.com/chef/chef-web-docs/blob/master/chef_master/source/api_delivery.rst>`__

.. tag chef_automate_mark

.. image:: ../../images/chef_automate_full.png
   :width: 40px
   :height: 17px

.. end_tag

The Chef Automate API is a REST API.

Authentication Methods
=====================================================

Authentication to the Chef Automate server occurs via a specific set of HTTP headers and two types of tokens:

* ``user token`` is a short-lived (seven days) token and can be obtained from the Chef Automate dashboard by entering this URL in your browser:
  
  .. code-block:: none

     https://YOUR_AUTOMATE_HOST/e/YOUR_AUTOMATE_ENTERPRISE/#/dashboard?token

* ``data_collector token`` is a long-lived token that can be set for your Chef Automate instance in ``/etc/delivery/delivery.rb``. Add ``data_collector['token'] = 'sometokenvalue'``, save your changes and then run ``sudo automate-ctl reconfigure``.

Required Headers
-----------------------------------------------------

The following authentication headers are required:

.. list-table::
   :widths: 200 300
   :header-rows: 1

   * - Feature
     - Description
   * - ``chef-delivery-enterprise``
     - .. tag api_chef_automate_headers_enterprise

       The name of the Chef Automate enterprise to use.

       .. end_tag

   * - ``chef-delivery-user``
     - .. tag api_chef_automate_headers_delivery_user

       The Chef Automate user to use for the API calls.

       .. end_tag

   * - ``chef-delivery-token``
     - .. tag api_chef_automate_headers_delivery_token

       The Chef Automate user token used in conjunction with ``chef-delivery-user``.

       .. end_tag

   * - ``x-data-collector-auth``
     - .. tag api_chef_automate_headers_data_collector_auth

       Set this to ``version=1.0`` in order to use the long-lived ``data_collector`` authentication instead of authenticating via ``chef-delivery-user`` and ``chef-delivery-token``.

       .. end_tag

   * - ``x-data-collector-token``
     - .. tag api_chef_automate_headers_data_collector_token

       The value of the ``data_collector`` token as set in ``/etc/delivery/delivery.rb`` if ``x-data-collector-auth`` is used.

       .. end_tag


The Chef Automate API is located at ``https://hostname`` and has the following endpoints:

API Endpoints
=====================================================


/api/_status
-----------------------------------------------------
The ``/api/_status`` endpoint can be used to check the health of the Chef Automate server without authentication. A Chef Automate instance may be configured as a standalone server or as a disaster recovery pair with primary and standby servers. The response from this endpoint depends on the type of configuration. This endpoint is located at ``/api/_status``.

**Request**

.. code-block:: xml

   GET /api/_status

This method has no request body.

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/api/_status"

**Response**

For a standalone server, the response will be similar to:

.. code-block:: json

   {
     "status": "pong",
     "configuration mode": "standalone",
     "upstreams": [
       {
         "postgres": {
           "status": "pong",
         },
         "lsyncd": {
           "status": "not_running",
         }
       }
     ]
   }

The top-level ``status`` value refers to the state of the core Chef Automate server only. It will return ``pong`` as long as the Chef Automate server is healthy even if there's a problem with one of the upstream systems; however, a response code of 500 will be returned in that case (as described in the response code section below).

.. note:: ``lsyncd`` should always report a status of ``not_running`` in a standalone configuration: any other value would indicate that it's configured when it shouldn't be (``lsync`` should only run on a disaster recovery primary).

For the primary server in a disaster recovery pair, the response will be similar to:

.. code-block:: json

   {
     "status": "pong",
     "configuration mode": "primary",
     "upstreams": [
       {
         "postgres": {
           "status": "pong",
           "standby_ip_address": "192.168.33.13",
           "pg_current_xlog_location": "0/3000D48"
         },
         "lsyncd": {
           "status": "pong",
           "latency": "0"
         }
       }
     ]
   }

In this configuration, the ``postgres`` and ``lsyncd`` upstreams will indicate the current state of disaster recovery replication.  For PostgreSQL, it will both indicate that it knows what the standby IP is supposed to be and the current ``location``. If the PostgreSQL replication is working correctly, it should match the value of the PostgreSQL ``xlog`` location reported by the standby (see below).

For ``lsyncd``, if the replication is up-to-date, ``latency`` should return 0; it may be above zero if changes have been queued up for replication, but it should quickly drop back down once the ``lsyncd`` server syncs changes (which should happen either after a fixed delay or when a certain number of changes have queued up). If it instead maintains a number above zero (or even continues to grow), that would indicate that there's an issue replicating git data in Chef Automate.

For the standby server in a disaster recovery pair, the response will be similar to:

.. code-block:: json

   {
     "status": "pong",
     "configuration mode": "cold_standby",
     "upstreams": [
       {
         "postgres": {
           "status": "pong",
           "pg_last_xlog_receive_location": "0/3000D48"
         },
         "lsyncd": {
            "status": "not_running",
         }
       }
     ]
   }

In this configuration, ``lsyncd`` should not be running; any other value would indicate a problem. For ``postgres``, if the replication is up-to-date, the ``location`` should match the value of the location on the primary it's replicating. If it's lagging (or behind and doesn't change), that would indicate an issue with PostgreSQL replication.

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - All services are OK. The response will show the service status as ``pong`` or ``not_running``. For example:

       .. code-block:: json

          {
            "status": "pong",
            "configuration mode": "standalone",
            "upstreams": [
              {
                "postgres": {
                  "status": "pong"
                },
                "lsyncd": {
                  "status": "not_running"
                }
              }
            ]
          }

   * - ``500``
     - One (or more) services are down. The response will show the service status as ``fail`` or ``degraded``. For example:

       .. code-block:: json

          {
            "status": "pong",
            "configuration mode": "cold_standby",
            "upstreams": [
              {
                "postgres": {
                "status": "fail",
                  "pg_last_xlog_receive_location": "0/3000D48"
              },
              "lsyncd": {
                "status": "not_running",
              }
            ]
          }

       For example, if replication is not running:

       .. code-block:: json

          {
            "status": "pong",
            "configuration mode": "primary",
            "upstreams": [
              {
                "postgres": {
                  "status": "degraded",
                  "replication": "fail",
                  "description": "Replication is not running. Check your configuration."
                },
                "lsyncd": {
                  "status": "pong",
                  "latency": "0"
                }
              }
            ]
          }

**Compliance Filters**
======================

+----------------+--------------------------------------------------+
| Name           | Filters search results based on scans that have: |
+================+==================================================+
|``start_time``  | end_times that are >= ``start_time``             |
+----------------+--------------------------------------------------+
|``end_time``    | end_times that are <= ``end_time``               |
+----------------+--------------------------------------------------+
|``environment`` | run in ``environment``                           |
+----------------+--------------------------------------------------+
|``node_id``     | run on target with ``node_id``                   |
+----------------+--------------------------------------------------+
|``platform``    | run on ``platform``                              |
+----------------+--------------------------------------------------+
|``profile_id``  | run against this ``profile_id``                  |
+----------------+--------------------------------------------------+

.. _compliance-market-api:

/compliance/market
-----------------------------------------------------
The Chef Automate server may store multiple compliance profiles.

The endpoint has the following methods: ``GET``.

GET (profiles)
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to get a list of compliance market profiles on the Chef Automate server.

This method takes no query parameters.

**Request**

.. code-block:: none

   GET /compliance/market/profiles

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/market/profiles" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

The response is similar to:

.. code-block:: json

    [
      {
        "name": "linux-baseline",
        "title": "DevSec Linux Security Baseline",
        "maintainer": "DevSec Hardening Framework Team",
        "copyright": "DevSec Hardening Framework Team",
        "copyright_email": "hello@dev-sec.io",
        "license": "Apache 2 license",
        "summary": "Test-suite for best-preactice Linux OS hardening",
        "version": "2.1.0",
        "supports": [
          {
            "os-family": "linux"
          }
        ],
        "depends": null
      },
      {
        "name": "postgres-baseline",
        "title": "Hardening Framework Postgres Hardening Test Suite",
        "maintainer": "DevSec Hardening Framework Team",
        "copyright": "DevSec Hardening Framework Team",
        "copyright_email": "hello@dev-sec.io",
        "license": "Apache 2 license",
        "summary": "Test-suite for best-practice postgres hardening",
        "version": "2.0.1",
        "supports": [
          {
            "os-family": "unix"
          }
        ],
        "depends": null
      },
      {
        "name": "ssh-baseline",
        "title": "DevSec SSH Baseline",
        "maintainer": "DevSec Hardening Framework Team",
        "copyright": "DevSec Hardening Framework Team",
        "copyright_email": "hello@dev-sec.io",
        "license": "Apache 2 license",
        "summary": "Test-suite for best-practice SSH hardening",
        "version": "2.2.0",
        "supports": [
          {
            "os-family": "unix"
          }
        ],
        "depends": null
      }
    ]

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.

GET (profile by name)
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to get the profile of a given NAME.

This method takes no query parameters.

**Request**

.. code-block:: none

   GET /compliance/market/profiles/NAME

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/market/profiles/linux-baseline" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

The response is similar to:

.. code-block:: json

   [
      {
         "name": "linux-baseline",
         "title": "DevSec Linux Security Baseline",
         "maintainer": "DevSec Hardening Framework Team",
         "copyright": "DevSec Hardening Framework Team",
         "copyright_email": "hello@dev-sec.io",
         "license": "Apache 2 license",
         "summary": "Test-suite for best-preactice Linux OS hardening",
         "version": "2.1.0",
         "supports": [
            {
               "os-family": "linux"
            }
         ],
         "depends": null
     }
   ]

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.

GET (profile by name & version)
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to get one specific VERSION of a profile of a given NAME.

This method takes no query parameters.

**Request**

.. code-block:: none

   GET /compliance/market/profiles/NAME/version/VERSION

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/market/profiles/linux-baseline/version/2.1.0" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

The response is similar to:

.. code-block:: json

   [
      {
         "name": "linux-baseline",
         "title": "DevSec Linux Security Baseline",
         "maintainer": "DevSec Hardening Framework Team",
         "copyright": "DevSec Hardening Framework Team",
         "copyright_email": "hello@dev-sec.io",
         "license": "Apache 2 license",
         "summary": "Test-suite for best-preactice Linux OS hardening",
         "version": "2.1.0",
         "supports": [
            {
               "os-family": "linux"
            }
         ],
         "depends": null
     }
   ]

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.

GET (profile tar by name)
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to get the latest version of a market profile tarball as specified by the NAME path parameter.

This method takes no query parameters.

**Request**

.. code-block:: none

   GET /compliance/market/profiles/NAME/tar

For example:

.. code-block:: bash

   curl -o linux-baseline.tar \
   "https://my-auto-server.test/compliance/market/profiles/linux-baseline/tar" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

TAR STREAM - download of the file requested (if it exists)


**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``404``
     - Not found. The requested profile was not found.

GET (profile tar by name & version)
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to get the market profile tarball for the given NAME and VERSION.

This method takes no query parameters.

**Request**

.. code-block:: none

   GET /compliance/market/profiles/NAME/version/VERSION/tar

For example:

.. code-block:: bash

   curl -o linux-baseline.tar \
   "https://my-auto-server.test/compliance/market/profiles/linux-baseline/version/2.1.0/tar" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

TAR STREAM - download of the file requested (if it exists)


**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``404``
     - Not found. The requested profile was not found.


.. _compliance-nodes-api:

/compliance/nodes
-----------------------------------------------------
Get the latest scan data for all nodes (or nodes that match filter(s)), then aggregate the compliance results from the
latest scans at the specified point in time.

The endpoint has the following methods: ``GET``.

GET (nodes)
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method returns aggregated compliance results across one or more nodes.

This method has the following optional parameters:

+-------------+------------+------------------------------------------------+
| Parameter   | Type       | Description                                    |
+=============+============+================================================+
| ``filters`` | string     | The search keywords, as well as any qualifiers.|
|             |            |                                                |
|             |            | - ``end_time``                                 |
|             |            | - ``start_time``                               |
|             |            |                                                |
+-------------+------------+------------------------------------------------+
| ``order``   | string     | The direction of the sort.                     |
|             |            | Can be either ``asc`` or ``desc``.             |
|             |            | Default: ``desc``                              |
+-------------+------------+------------------------------------------------+
| ``page``    | string     | page number for paginated data                 |
+-------------+------------+------------------------------------------------+
| ``per_page``| string     | items per page                                 |
+-------------+------------+------------------------------------------------+
| ``sort``    | string     | *What to sort results by. Can be any of        |
|             |            | the following:*                                |
|             |            |                                                |
|             |            | - ``environment``                              |
|             |            | - ``latest_report.controls.failed.critical``   |
|             |            | - ``latest_report.controls.failed.total``      |
|             |            | - ``latest_report.end_time``                   |
|             |            | - ``latest_report.status``                     |
|             |            | - ``name``                                     |
|             |            | - ``platform``                                 |
|             |            | - ``status``                                   |
+-------------+------------+------------------------------------------------+





**Request**

.. code-block:: none

   GET /compliance/nodes

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/nodes" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

The response is similar to:

.. code-block:: json

   [
     {
       "id": "74a54a28-c628-4f82-86df-61c43866db6a",
       "name": "teal-spohn",
       "platform": {
         "name": "centos"
       },
       "environment": "DevSec Prod Alpha",
       "latest_report": {
         "id": "3ca95021-84c1-43a6-a2e7-be10edcb238d",
         "end_time": "2017-04-04T10:18:41+01:00",
         "status": "failed",
         "controls": {
           "total": 113,
           "passed": {
             "total": 22
           },
           "skipped": {
             "total": 68
           },
           "failed": {
             "total": 23,
             "minor": 0,
             "major": 0,
             "critical": 23
           }
         }
       }
     },
     {
       "id": "99516108-8126-420e-b03e-a90a52f25751",
       "name": "red-brentwood",
       "platform": {
         "name": "debian"
       },
       "environment": "DevSec Prod Zeta",
       "latest_report": {
         "id": "44024b50-2e0d-42fa-a57c-25e05e48a1b5",
         "end_time": "2017-03-06T09:18:41Z",
         "status": "failed",
         "controls": {
           "total": 59,
           "passed": {
             "total": 23
           },
           "skipped": {
             "total": 14
           },
           "failed": {
             "total": 22,
             "minor": 0,
             "major": 0,
             "critical": 22
           }
         }
       }
     }
   ]


**Response Codes**

.. list-table::
   :widths: 100 420
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``400``
     - Bad Request. Something is wrong with the request. Client should look closely at the request they're making.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``500``
     - Internal Server Error. Problem on the backend.

GET (node by name)
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to get the profile of a given NAME.

This method takes no query parameters.

**Request**

.. code-block:: none

   GET /compliance/nodes/NODE

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/nodes/74a54a28-c628-4f82-86df-61c43866db6a" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

The response is similar to:

.. code-block:: json

   {
     "id": "74a54a28-c628-4f82-86df-61c43866db6a",
     "name": "teal-spohn",
     "platform": {
       "name": "centos",
       "release": "5.11"
     },
     "environment": "DevSec Prod Alpha",
     "latest_report": {
       "id": "3ca95021-84c1-43a6-a2e7-be10edcb238d",
       "end_time": "0001-01-01T00:00:00Z",
       "status": "failed",
       "controls": {
         "total": 113,
         "passed": {
           "total": 22
         },
         "skipped": {
           "total": 68
         },
         "failed": {
           "total": 23,
           "minor": 0,
           "major": 0,
           "critical": 23
         }
       }
     },
     "profiles": [
       {
         "name": "linux-baseline",
         "version": "2.0.1",
         "id": "b53ca05fbfe17a36363a40f3ad5bd70aa20057eaf15a9a9a8124a84d4ef08015"
       },
       {
         "name": "ssh-baseline",
         "version": "2.1.1",
         "id": "3984753145f0db693e2c6fc79f764e9aff78d892a874391fc5f5cc18f4675b68"
       }
     ]
   }

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``400``
     - Bad Request. Something is wrong with the request. Client should look closely at the request they're making.
   * - ``404``
     - Not Found. The resource was not found.
   * - ``500``
     - Internal Server Error. Problem on the backend.

.. _compliance-profile-api:

/compliance/profiles/OWNER
-----------------------------------------------------
The Chef Automate server may store multiple compliance profiles, namespaced by owners.

The endpoint has the following methods: ``GET`` and ``POST``.

GET
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to get a list of compliance profiles namespaced by OWNER on the Chef Automate server.

This method has no parameters.

**Request**

.. code-block:: none

   GET /compliance/profiles/OWNER

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/profiles/john" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

The response is similar to:

.. code-block:: none

   {
     "linux": {
       "id": "linux",
       "name": "linux",
       "title": "Basic Linux",
   ...
   }

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``404``
     - Not Found. The OWNER specified in the request was not found.


POST
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``POST`` method is used to upload a compliance profile(as a tarball) namespaced by OWNER.

This method has no parameters.

**Request**

.. code-block:: none

   POST /compliance/profiles/OWNER

For example:

.. code-block:: bash

   tar -cvzf /tmp/newprofile.tar.gz /home/user/newprofile
   curl -X POST "https://my-auto-server.test/compliance/profiles/john" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..." \
   --form "file=@/tmp/newprofile.tar.gz"

**Response**

No Content

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``500``
     - Internal Error. Profile check failed.


/compliance/profiles/OWNER/PROFILE
-----------------------------------------------------
Endpoint targeting specific compliance profile.

The following methods are available: ``GET`` and ``DELETE``.

GET
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` method is used to return details of a particular profile.

This method has no parameters.

**Request**

.. code-block:: none

   GET /compliance/profiles/OWNER/PROFILE

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/profiles/john/linux" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

The response is similar to:

.. code-block:: none

   {
     "id": "linux",
     "owner": "john",
     "name": "linux",
     "title": "Basic Linux",
       "controls": {
        "basic-1": {
   ...
   }

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``404``
     - Not Found. The OWNER specified in the request was not found.


DELETE
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``DELETE`` method is used to remove a particular profile.

This method has no parameters.

**Request**

.. code-block:: none

   DELETE /compliance/profiles/OWNER/PROFILE

For example:

.. code-block:: bash

   curl -X DELETE "https://my-auto-server.test/compliance/profiles/john/linux" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..."

**Response**

No Content

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``404``
     - Not Found. The OWNER or PROFILE specified in the request was not found.


/compliance/profiles/OWNER/PROFILE/tar
-----------------------------------------------------

GET
+++++++++++++++++++++++++++++++++++++++++++++++++++++
The ``GET`` is used to download a profile as a tarball.

This method has no parameters.

**Request**

.. code-block:: none

   GET /compliance/profiles/OWNER/PROFILE/tar

For example:

.. code-block:: bash

   curl -X GET "https://my-auto-server.test/compliance/profiles/john/linux" \
   -H "chef-delivery-enterprise: acme" \
   -H "chef-delivery-user: john" \
   -H "chef-delivery-token: 7djW35..." > /tmp/profile.tar.gz

**Response**

TAR STREAM

**Response Codes**

.. list-table::
   :widths: 100 400
   :header-rows: 1

   * - Response Code
     - Description
   * - ``200``
     - OK. The request was successful.
   * - ``401``
     - Unauthorized. The user who made the request is not authorized to perform the action.
   * - ``404``
     - Not Found. The OWNER or PROFILE specified in the request was not found.
