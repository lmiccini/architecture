# Distributed Zones with BGP, Third Party Storage, and TNA (Two-Node with Arbiter)

This Deployed Topology (DT) is a variant of [dz-storage](../dz-storage)
adapted for a Two-Node with Arbiter (TNA) OpenShift cluster topology.
TNA uses 2 full control plane nodes + 1 lightweight arbiter node (etcd only)
to provide quorum across 2 datacenters plus a minimal arbiter site.

## Topology

- **2 data zones + 1 arbiter zone:**
  - zone A (DC-A): ocp-worker-0, ocp-worker-1, ocp-worker-2, ocp-master-0
  - zone B (DC-B): ocp-worker-3, ocp-worker-4, ocp-worker-5, ocp-master-1
  - zone C (arbiter): ocp-worker-6 (quorum voter only), TNA arbiter node (etcd only)
- **EDPM nodes (2 data zones only):**
  - zone A: r0-compute-0, r0-compute-1, r0-networker-0, leaf-0, leaf-1
  - zone B: r1-compute-0, r1-compute-1, r1-networker-0, leaf-2, leaf-3
- **OpenStack availability zones:** az0 and az1 only (no az2)

## Key differences from dz-storage

| Aspect | dz-storage | dz-storage-tna |
|--------|-----------|----------------|
| OCP masters | 3 full | 2 full + 1 TNA arbiter |
| OCP workers | 9 (3 per zone) | 7 (3+3+1) |
| EDPM computes | 6 (2 per zone) | 4 (2 per data DC) |
| EDPM networkers | 3 (1 per zone) | 2 (1 per data DC) |
| OpenStack AZs | 3 (az0, az1, az2) | 2 (az0, az1) |
| Cinder volumes | 3 backends | 2 backends |
| Glance stores | 3 edge + 1 default | 2 edge + 1 default |
| Manila shares | 3 backends | 2 backends |
| Galera replicas | 3 | 3 (quorum across 3 zones) |
| RabbitMQ replicas | 3 | 3 (quorum across 3 zones) |

## Design rationale

The arbiter zone (zoneC) hosts:
- 1 Galera pod (quorum voter for database)
- 1 RabbitMQ pod (quorum voter for messaging)
- 1 Memcached pod (local cache)

It does NOT host storage backends (cinder-volume, glance, manila-share)
or compute nodes. This keeps the arbiter site lightweight while ensuring
all quorum-based services survive the loss of any single zone.

[Topology CRDs](https://github.com/openstack-k8s-operators/infra-operator/pull/325) are
used to either spread pods across zones or keep them within a zone.

## Prerequisites

- OpenShift cluster deployed with TNA topology (2 masters + 1 arbiter).
  See [dev-scripts](https://github.com/openshift-metal3/dev-scripts/) with
  `AGENT_E2E_TEST_SCENARIO=TNA_IPV4` or `NUM_MASTERS=2 NUM_ARBITERS=1`.

- Self Node Remediation and Node Health Checks (chapters 2 and 6 from
  [Workload Availability](https://docs.redhat.com/en/documentation/workload_availability_for_red_hat_openshift/24.4/html-single/remediation_fencing_and_maintenance)).

- A storage class has been created (e.g. LVMS).

- Storage arrays physically located in each data zone (az0, az1).

## Stages

All stages must be executed in the order listed below.

1. [Configure taints on the OCP worker](configure-taints.md)
2. [Disable RP filters on OCP nodes](disable-rp-filters.md)
3. [Install the OpenStack K8S operators and their dependencies](../../common/)
4. [Apply metallb customization required to run a speaker pod on the OCP tester node](metallb/)
5. [Define Zones and Topologies](topology/)
6. [Configure networking and deploy the OpenStack control plane with storage](control-plane.md)
7. [Create BGPConfiguration after control plane is deployed](bgp-configuration.md)
8. [Configure and deploy the dataplane - networker and compute nodes](data-plane.md)
9. [Validate Distributed Zone Storage](validate.md)
