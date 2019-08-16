# Ansible Cisco Network Assurance Engine Playbooks

Cisco Network Assurance Engine (NAE) transforms operations in data center networks to a fundamentally more proactive model. Built on Cisco’s patented network verification technology, it is the most comprehensive assurance engine that mathematically verifies the entire network for correctness, giving operators confidence that their network is always operating consistent with intent.

## Resources

DevNet Sandbox:
- [Cisco Network Assurance Engine](https://devnetsandbox.cisco.com/RM/Diagram/Index/fd4efc55-c723-4be9-89e6-950daf0bba31?diagramType=Topology)
- [Cisco Network Assurance Engine 3.1 v1](https://dcloud2-rtp.cisco.com/content/instantdemo/cisco-network-assurance-engine-3-1-v1?returnPathTitleKey=content-view) **dCloud Demo** Please read the **Caveats** section below

## Prerequisites

- None

## Dependancies

- Ansible 2.8.x
- Python 3.7.x

## Configuration / Getting Started

These playbooks use the Ansible `uri` module so there's no special requirements. They've been tested with `2.7.10`, NAE 3.1(1) and the [DevNet API Documentation](https://developer.cisco.com/docs/network-assurance-engine/#!introduction).

First use [nae_api_login.yml](nae_api_login.yml) to set up the session tokens. It requires setting some NAE specific headers together with a session cookie (not obvious in the docs so worth pointing out).

A user can only be logged in to the Web/GUI OR the API at one time. I had a lot of trouble with expired sessions until I figured it out. If you need to be using the GUI at the same time, then get the CSRF token and cookies from the GUI session manually!

If using the **DevNet Sandbox** (https://nae-sandbox.cisco.com) or any NAE that can be connected to directly, that's everything.

If using the **DCloud Demo**, there is an additional layer of DCloud cookies that are required and the easiest way is to retrieve everything manually from the browser session after logging into the NAE app and create a separate `vars/session-dcloud.yml` file.

To fetch the CSRF token from a running web browser, in Chrome you can type `localStorage.CSRF_TOKEN` in the developer console. Reload any page with the Network tab open and select a resource to see all the other cookies.

## The workflow

All example output here is from the DevNet Sandbox.

First, run `ansible-playbook nae_aci_fabrics.yml` to get the list of configured fabrics.

```
Name                          | Operational Mode    | Status         | Fabric ID                                    | APICs               
------------------------------+---------------------+----------------+----------------------------------------------+---------------------
aci-k8tes-sandbox             | ONLINE              | RUNNING        | d9e15b75-19402c77-0f0a-41d8-9ec6-b5409e60cb96|                     
                              |                     |                |                                              | 10.10.20.12         
------------------------------+---------------------+----------------+----------------------------------------------+---------------------
```

Then get all the epochs for a fabric by running `ansible-playbook nae_fabric_epochs.yml -e "fabric_id=d9e15b75-19402c77-0f0a-41d8-9ec6-b5409e60cb96"`. Note the `fabric_id` is defined in the playbook and we're overriding it here - there are a few other parameters in the playbook that can be tweaked.

```
Analysis Time            | Status    | Epoch ID                                     | Events                   
-------------------------+-----------+----------------------------------------------+----------------         
2019-08-10T13:22:09Z     | FINISHED  | d9e15b75-8792cd91-0e2d-355d-9a03-f28ce554ed54|                          
                         |           |                                              | info: 391                
                         |           |                                              | critical: 10             
                         |           |                                              | major: 43                
                         |           |                                              | minor: 3                 
                         |           |                                              | warning: 108             
-------------------------+-----------+----------------------------------------------+----------------         
2019-08-10T13:07:09Z     | FINISHED  | d9e15b75-302611e9-8fdf-332c-a21a-1b549d5e5d16|                          
                         |           |                                              | info: 391                
                         |           |                                              | critical: 10             
                         |           |                                              | major: 43                
                         |           |                                              | minor: 3                 
                         |           |                                              | warning: 108             
-------------------------+-----------+----------------------------------------------+----------------         
... output snipped
```

Then get all the events for a specific epoch by running `ansible-playbook nae_epoch_events.yml -e "epoch_id=d9e15b75-8792cd91-0e2d-355d-9a03-f28ce554ed54"`. Note the `epoch_id` is defined in the playbook and we're overriding it here - there are a few other parameters in the playbook that can be tweaked (most notably the severity category).

```
... output snipped

EPOCH ID: d9e15b75-8792cd91-0e2d-355d-9a03-f28ce554ed54
SEVERITY: EVENT_SEVERITY_CRITICAL

Event                                                       | Category                 | Subcategory              
============================================================+==========================+==========================
CONNECTED_EP_LEARNING_ERROR                                 | TENANT_ENDPOINT          | ENDPOINT_LEARNING        
------------------------------------------------------------+--------------------------+--------------------------
Description: Endpoint information is not consistent across the fabric leafs and spines.
ID: d9e15b75-8792cd91-0e2d-355d-9a03-f28ce554ed54-3980ec56208a822d98d148b99de73507
============================================================+==========================+==========================
ENFORCED_VRF_POLICY_VIOLATION                               | TENANT_SECURITY          | VRF_SECURITY             
------------------------------------------------------------+--------------------------+--------------------------
Description: VRF is in enforced mode. APIC policy for implicit deny log is not enforced on Leaf hardware.
ID: d9e15b75-8792cd91-0e2d-355d-9a03-f28ce554ed54-98d835f0689ef00e15a3782bb093f55b
============================================================+==========================+==========================

```

And, finally, we can get event details by querying for a specific event_id: `ansible-playbook nae_event_details.yml -e "event_id=d9e15b75-8792cd91-0e2d-355d-9a03-f28ce554ed54-98d835f0689ef00e15a3782bb093f55b"`. Worth noting the event details data model is quite large and deeply nested, we extract only a subset of metadata here and more can be done by inspecting the JSON object as needed.

```
-------------------------+-----------+----------------------------------------------+----------------         
Primary Affected Object:
  NAME: sbx_shared
  TYPE: CANDID_OBJECT_TYPE_VRF
  ID: uni/tn-common/ctx-sbx_shared

ACI Fabric: aci-k8tes-sandbox
Description: VRF is in enforced mode. APIC policy for implicit deny log is not enforced on Leaf hardware.
Impact: Unauthorized traffic will be allowed in this VRF.
ENFORCED_VRF_POLICY_VIOLATION/TENANT_SECURITY/VRF_SECURITY/EVENT_SEVERITY_CRITICAL
-------------------------+-----------+----------------------------------------------+----------------
```

All generated reports can be found in the [output](output) folder and I've included the ones from the above run into the repository.


## Caveats:

If using the [Cisco Network Assurance Engine 3.1 v1](https://dcloud2-rtp.cisco.com/content/instantdemo/cisco-network-assurance-engine-3-1-v1?returnPathTitleKey=content-view) **dCloud Demo**, there is an additional layer of dCloud cookies that are required and the easiest way is to retrieve everything manually from the browser session after logging into the NAE app and create a separate `vars/session-dcloud.yml` file.

To fetch the CSRF token from a running web browser, in Chrome you can type `localStorage.CSRF_TOKEN` in the developer console. Reload any page with the Network tab open and select a resource to see all the other cookies.
