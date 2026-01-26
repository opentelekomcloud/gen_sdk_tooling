# Research: opentelekomcloud-docs Structure

The **opentelekomcloud-docs** organization on GitHub contains ~100+ repositories, one per OTC service.

## Organization

- GitHub: https://github.com/opentelekomcloud-docs
- Template: https://github.com/opentelekomcloud-docs/template

## Repository Structure

```
service-name/
├── api-ref/
│   └── source/             # API Reference (RST)
│       ├── index.rst
│       ├── apis_recommended/
│       │   └── resource_management/
│       │       └── operation.rst
│       └── common/
│           ├── error_codes.rst
│           └── returned_values.rst
├── umn/
│   └── source/             # User Manual (RST)
├── doc/                    # Main documentation
├── requirements.txt
├── setup.cfg
├── tox.ini
├── zuul.yaml               # CI OpenTelekomCloud
└── README.rst
```

### Key Repositories

- ECS: https://github.com/opentelekomcloud-docs/elastic-cloud-server
- CCE: https://github.com/opentelekomcloud-docs/cloud-container-engine
- VPC: https://github.com/opentelekomcloud-docs/virtual-private-cloud

---

## RST File Format

Each endpoint is documented in a separate RST file with **standard sections**:

```rst
:original_name: en-us_topic_0110472767.rst

.. _en-us_topic_0110472767:

Querying Clusters
=================

Function
--------
This API is used to query all clusters in a project.

URI
---
GET /v3/{project_id}/clusters

.. table:: **Table 1** Path parameters

   +--------------+-----------+--------+---------------------------+
   | Parameter    | Mandatory | Type   | Description               |
   +==============+===========+========+===========================+
   | project_id   | Yes       | String | Specifies the project ID. |
   +--------------+-----------+--------+---------------------------+

Request
-------
.. table:: **Table 2** Request header parameters

   +--------------+-----------+--------+---------------------------+
   | Parameter    | Mandatory | Type   | Description               |
   +==============+===========+========+===========================+
   | X-Auth-Token | Yes       | String | User token.               |
   +--------------+-----------+--------+---------------------------+

Response
--------
.. table:: **Table 3** Response body parameters

   +-----------+------------------+---------------------------+
   | Parameter | Type             | Description               |
   +===========+==================+===========================+
   | clusters  | Array of objects | List of cluster objects.  |
   +-----------+------------------+---------------------------+

Example Request
---------------
.. code-block:: text

   GET https://{endpoint}/v3/{project_id}/clusters

Example Response
----------------
.. code-block:: json

   {
     "clusters": [
       {
         "id": "cluster-uuid",
         "name": "my-cluster",
         "status": "Available"
       }
     ]
   }
```

---

## Parsing Patterns

RST files contain **structured information** that can be extracted:

| Section | Extracted Data |
|---------|----------------|
| URI | HTTP method, path template, path parameters |
| Table "Path/Query parameters" | Name, mandatory flag, type, description |
| Table "Request body" | Request body structure |
| Table "Response body" | Response structure |
| Example Request/Response | Validation data |

---

## Edge Cases

### OpenStack APIs Inside OTC Docs

Some services contain **nested OpenStack sections**:

```
elastic-cloud-server/
└── api-ref/
    └── source/
        ├── apis/                    # OTC-specific APIs
        └── openstack_nova_apis/     # Standard OpenStack Nova APIs
```

**Affected services:**
- ECS (Elastic Cloud Server) → OpenStack Nova
- EVS (Elastic Volume Service) → OpenStack Cinder
- VPC → OpenStack Neutron
- RDS → OpenStack Trove (partially)

The parser must mark such endpoints as `source: "openstack"` in the IR.
If an API is marked as source: openstack, the generator should attempt to 
inherit from the corresponding class in openstack-sdk instead of generating a base resource from scratch.

### API Versions

Inside `api-ref/source` there may be version subdirectories:

```
api-ref/source/
├── v1/
│   └── clusters.rst
├── v2/
│   └── clusters.rst
└── v3/
    └── clusters.rst
```

The parser must extract the API version and include it in the IR.

---

## Consistency Analysis

**High consistency:**
- ✅ Repository structure (api-ref/source, umn/source)
- ✅ RST file sections (Function, URI, Request, Response, Examples)
- ✅ Parameter table format
- ✅ Topic ID convention (`en-us_topic_XXXXXXXXXX`)
- ✅ All repositories created from single template

**Variations:**
- ⚠️ API versions (`/v1/`, `/v2/`, `/v3/`)
- ⚠️ Subfolder organization (depends on service complexity)
- ⚠️ Native vs Recommended API separation
- ⚠️ OpenStack API sections (for ECS, EVS, VPC)

---

## Key Takeaways

- RST format is **well-structured** and suitable for automatic parsing
- Parameter tables contain **all necessary metadata** (type, mandatory, description)
- Request/response examples can be used for **validation** of generated SDK
- Structural consistency facilitates creating a universal parser
- Need to account for **OpenStack API sections** and **API versions**