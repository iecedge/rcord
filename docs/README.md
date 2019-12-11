# R-CORD Service

The RCORD Service represents the `Subscriber` in the service chain. This service is always located at the beginning of a chain.

## Models

The R-CORD service has the following three models:

- `RCORDService`. In addition to the standard Service model fields such as `name`, adds the following R-CORD-specific fields:
    - `access`. Type of access service. Currently the only usable value is "voltha".
- `RCORDSubscriber`. This model extends `ServiceInstance` and holds several subscriber-related fields:
    - `creator`. The user that created the subscriber. Data plane services that implement compute resources may be able to leverage the `creator` field to account for ownership of those compute resources.
    - `status`. [`enabled` | `disabled` | `pre-provisioned` | `awaiting-auth` | `auth-failed`]. The status of the subscriber, often determined by the workflow driver.
    - `c_tag`, `s_tag`. VLAN tags associated with this subscriber.
    - `onu_device`. Serial number of subscriber's ONU. Must match an ONU Device in an access service.
    - `mac_address`. MAC address of subscriber.
    - `tech_profile_id`. This together with the `technology` of the ONU associated with the subscriber must match the `profile_id` and `technology` of a `TechnologyProfile` object from the `olt-service` service.
    - `nas_port_id`.
    - `circuit_id`.
    - `remote_id`.
    - `upstream_bps`. Bandwidth profile for upstream.
    - `downstream_bps`. Bandwidth profile for downstream.
- `RCORDIpAddress`. Holds an IP address that is associated with a subscriber. These are typically created by the workflow driver, when it handles DHCP messages.
    - `subscriber`. Relation to the subscriber that this IP address applies to.
    - `ip`. IP Address.
    - `description`. A short description of the IP Address.
- `BandwidthProfile`. Holds a bandwidth profile.
    - `name`. Name of this bandwidth profile.
    - `cir`. Committed information rate.
    - `cbs`. Committed burst size.
    - `eir`. Excess information rate.
    - `ebs`. Excess burst size.
    - `air`. Access information rate.



## Example Tosca - Create a Subscriber

The following TOSCA recipe creates:

- `RCORDSubscriber`
- `BandwidthProfile`

```yaml
tosca_definitions_version: tosca_simple_yaml_1_0
imports:
  - custom_types/rcordsubscriber.yaml
  - custom_types/bandwidthprofile.yaml

description: Pre-provsion a subscriber

topology_template:
  node_templates:

    # Pre-provision the subscriber
    high_speed_bp:
      type: tosca.nodes.BandwidthProfile
      properties:
         air: 2000
         cbs: 2000
         cir: 2000
         ebs: 2000
         eir: 2000
         name: Bronze

    # Pre-provision the subscriber the subscriber
    onf_subscriber_1:
      type: tosca.nodes.RCORDSubscriber
      properties:
        name: Sub_ALPHe3d1cfde
        status: pre-provisioned
        c_tag: 222
        s_tag: 111
        onu_device: ALPHe3d1cfde
        nas_port_id : "PON 1/1/04/1:1.1.1"
        circuit_id: foo2
        remote_id: bar2
        tech_profile_id: 64
      requirements:
        - upstream_bps:
            node: high_speed_bp
            relationship: tosca.relationships.BelongsToOne
        - downstream_bps:
            node: high_speed_bp
            relationship: tosca.relationships.BelongsToOne
```

> NOTE: an `onu_device` with the provided serial number must exist in the system.
> For more information about ONU Devices, please refer to the
> [vOLTService](../olt-service/README.md) guide.

## Integration with other Services

The R-CORD Service has no western neighbors, as it is always the root of a subscriber service chain. It will have a westbound neighbor, which is typically an access service.

If the R-CORD Service is configured with `access=voltha`, the following requirements apply:

- There are two `provider_service` linked to the R-CORD Servioce.
- First one is `VOLT_SERVICE` which exposes an API called `has_access_device(onu_serial_number)`
  that returns a boolean. This is used to validate that the ONU the subscriber
  is pointing to really exists.
- The `VOLT_SERVICE` also exposes API called `get_olt_technology_from_unu_sn(onu_serial_number)` and `get_tech_profile(technology, tech_profile_id)`
  that returns a boolean. This is used to validate that the Technology Profile the subscriber
  is pointing to really exists. See [Technology Profile Management](https://github.com/opencord/voltha/tree/master/common/tech_profile) for more informations.
- Second service is `ONOS_SERVICE` which adds a rest endpoint to the Sadis application in order to flush the cache for a single subscriber.This provides an endpoint
  to flush the cache for a single subscriber as well as flush the entire cache.
## Synchronizer workflow

The R-CORD Service synchronizer implements no sync steps as it does not directly interact with any external components. It does implement one model policy.

### RCORDSubscriberPolicy

The policy manages the service chain associated with the subscriber. If the subscriber is in `pre-provisioned` status, then no work is done. Otherwise the policy attempts to bring the service chain into compliance with the status field. Statuses of `enabled` are allocated a service chain. Statuses of `awaiting-auth`, `auth-failed`, or `disabled` are prevented from having a service chain.

The service chain typically proceeds eastbound from the RCORDSubscriber to an access service instance, such as VOLTServiceInstance.



