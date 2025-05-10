# Product Requirements Document: IPAM InCluster Network Address Support for TalosConfig

## Document Information

- **Title**: IPAM InCluster Network Address Support for TalosConfig
- **Status**: Implementation
- **Version**: 1.0
- **Date**: 20250510
- **Authors**: Erik Nelson

## Executive Summary

This document outlines the implementation strategy for enhancing the Cluster API Bootstrap Provider Talos (CABPT) to support dynamically allocated IP addresses from the IPAM InCluster provider. This feature will enable infrastructure where a node's network configuration must be set via Talos machine configuration rather than relying on DHCP.

The implementation will allow a `TalosConfig` to wait for the IPAM provider to allocate an IP address to a Machine, and then use that IP address in the generated Talos machine configuration. This addresses a current limitation where users need to hardcode IP addresses in their templates, preventing multi-node deployments with the same configuration template.

## Background

### Current Behavior

Currently, the CABPT creates Talos machine configurations without waiting for IP addresses to be allocated by IPAM providers. When using the IPAM InCluster provider, the IP allocation information is written to the `Machine.status.addresses` field. However, this information isn't used during the TalosConfig generation process.

Users have attempted to use Go templating syntax to extract IP addresses:

```yaml
machine:
  network:
    interfaces:
      - interface: eth0
        addresses:
          - {{ index (where .Machine.Status.Addresses "type" "InternalIP") 0 "address" }}/24
```

However, this template string is literally placed in the Talos machine config, indicating that the bootstrap provider doesn't support dynamic templating.

### Proposed Solution

We propose to enhance the TalosConfig controller to:

1. Wait for IP addresses to be allocated by the IPAM provider (indicated by the Machine.status.addresses field)
2. Extract the allocated IP address from Machine.status.addresses
3. Generate a network configuration in the Talos machine configuration using the allocated IP address

This will be implemented through a new feature in the TalosConfigSpec called `ipAddress` which allows users to specify how they want to consume IP addresses from Machine status.

## Requirements

### Functional Requirements

1. **IP Address Source Configuration**:
   - Add a new `IPAddressSpec` type to define IP address sources, with support for "MachineStatus" as the initial source type.
   - Support specifying which address type to use from Machine.status.addresses (e.g., "InternalIP").
   - Support configuring network interface name, subnet prefix, and default gateway.

2. **Delayed Configuration Generation**:
   - When configured, the TalosConfig controller must wait for the Machine's IP address to be allocated by the IPAM provider before generating the machine configuration.
   - The controller should periodically requeue until an IP address is available.

3. **Dynamic IP Configuration**:
   - The controller must extract the IP address from the Machine's status and incorporate it into the generated machine configuration.
   - Support configuring the network interface with DHCP disabled and the static IP address.
   - Support adding default gateway and route configuration.

4. **User Experience**:
   - Provide clear documentation on how to use this feature.
   - Ensure backward compatibility with existing configurations.

### Non-Functional Requirements

1. **Performance**:
   - The controller should use reasonable polling intervals (e.g., 10 seconds) when waiting for IP allocation.
   - Minimize unnecessary reconciliations.

2. **Compatibility**:
   - The implementation must work with existing CAPI and IPAM providers.
   - The feature should work with Talos Linux versions v1.2 and newer.

3. **Maintainability**:
   - Code changes should follow project patterns and conventions.
   - Adequate test coverage must be provided.

## Detailed Design

### API Changes

#### 1. New Types in `api/v1alpha3/types.go`

```go
// IPAddressSource is the definition of IP address source.
type IPAddressSource string

// IPAddressSourceMachineStatus sets the IP address from Machine status addresses.
const IPAddressSourceMachineStatus IPAddressSource = "MachineStatus"

// IPAddressSpec defines the IP address source and configuration.
type IPAddressSpec struct {
    // Source of the IP address.
    //
    // Allowed values:
    // "MachineStatus" (use the IP address allocated by IPAM provider).
    Source IPAddressSource `json:"source,omitempty"`

    // AddressType specifies which type of address to use from the Machine status.
    // Usually "InternalIP" for IPAM-allocated addresses.
    AddressType string `json:"addressType,omitempty"`

    // Interface is the name of the network interface to configure.
    Interface string `json:"interface,omitempty"`

    // SubnetPrefix is the subnet prefix (CIDR notation without IP, e.g., "/24").
    SubnetPrefix string `json:"subnetPrefix,omitempty"`

    // DefaultGateway is the gateway IP address.
    DefaultGateway string `json:"defaultGateway,omitempty"`
}
```

#### 2. Update to `api/v1alpha3/talosconfig_types.go`

```go
// TalosConfigSpec defines the desired state of TalosConfig
type TalosConfigSpec struct {
    TalosVersion     string          `json:"talosVersion,omitempty"`
    GenerateType     string          `json:"generateType"`
    Data             string          `json:"data,omitempty"`
    ConfigPatches    []ConfigPatches `json:"configPatches,omitempty"`
    StrategicPatches []string        `json:"strategicPatches,omitempty"`
    Hostname         HostnameSpec    `json:"hostname,omitempty"`
    // IPAddress defines how to configure network interfaces with IP addresses.
    IPAddress        IPAddressSpec   `json:"ipAddress,omitempty"`
}
```

### Controller Changes

#### 1. New Method to Check IP Allocation

```go
// hasMachineIPAllocated checks if the Machine has an IP address allocated.
func (r *TalosConfigReconciler) hasMachineIPAllocated(ctx context.Context, scope *TalosConfigScope) (bool, string, error) {
    // Only check for IP if configured to use MachineStatus as source
    if scope.Config.Spec.IPAddress.Source != v1alpha3.IPAddressSourceMachineStatus {
        return true, "", nil
    }

    // Only applies to Machines, not MachinePools
    if scope.ConfigOwner.IsMachinePool() {
        return true, "", nil
    }

    // Get the Machine object
    machine := &capiv1.Machine{}
    if err := runtime.DefaultUnstructuredConverter.FromUnstructured(scope.ConfigOwner.Object, machine); err != nil {
        return false, "", err
    }

    // Get AddressType to look for, default to InternalIP if not specified
    addressType := scope.Config.Spec.IPAddress.AddressType
    if addressType == "" {
        addressType = "InternalIP"
    }

    // Look for an address of the specified type
    for _, addr := range machine.Status.Addresses {
        if string(addr.Type) == addressType && addr.Address != "" {
            return true, addr.Address, nil
        }
    }

    // No IP address found
    return false, "", nil
}
```

#### 2. Update to Reconcile Method

```go
func (r *TalosConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (_ ctrl.Result, rerr error) {
    // ... existing code ...

    // If configured to use machine IP address, check if it's allocated
    if scope.Config.Spec.IPAddress.Source == v1alpha3.IPAddressSourceMachineStatus {
        hasIP, _, err := r.hasMachineIPAllocated(ctx, tcScope)
        if err != nil {
            return ctrl.Result{}, err
        }

        if !hasIP {
            log.Info("Waiting for IP address to be allocated by the IPAM provider")
            // Requeue after 10 seconds to check again
            return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
        }
    }

    // Continue with the rest of the reconciliation
    // ... existing code ...
}
```

#### 3. Update to reconcileGenerate Method

```go
func (r *TalosConfigReconciler) reconcileGenerate(ctx context.Context, tcScope *TalosConfigScope) error {
    // ... existing code ...

    // After retData is created but before returning:
    if !tcScope.ConfigOwner.IsMachinePool() && tcScope.Config.Spec.IPAddress.Source == v1alpha3.IPAddressSourceMachineStatus {
        hasIP, ipAddress, err := r.hasMachineIPAllocated(ctx, tcScope)
        if err != nil {
            return err
        }

        if !hasIP {
            return fmt.Errorf("no IP address allocated yet")
        }

        // Create network configuration patch
        interfaceName := tcScope.Config.Spec.IPAddress.Interface
        if interfaceName == "" {
            interfaceName = "eth0" // Default interface name
        }

        subnetPrefix := tcScope.Config.Spec.IPAddress.SubnetPrefix
        if subnetPrefix == "" {
            subnetPrefix = "/24" // Default subnet
        }

        // Apply IP address via strategic patch
        networkPatch := fmt.Sprintf(`
machine:
  network:
    interfaces:
      - interface: %s
        dhcp: false
        addresses:
          - %s%s`, interfaceName, ipAddress, subnetPrefix)

        // Add gateway if specified
        if gateway := tcScope.Config.Spec.IPAddress.DefaultGateway; gateway != "" {
            networkPatch += fmt.Sprintf(`
        routes:
          - network: 0.0.0.0/0
            gateway: %s`, gateway)
        }

        // Apply the patch
        patch, err := configpatcher.LoadPatch([]byte(networkPatch))
        if err != nil {
            return fmt.Errorf("failure loading network patch: %w", err)
        }

        out, err := configpatcher.Apply(configpatcher.WithBytes([]byte(retData.BootstrapData)), []configpatcher.Patch{patch})
        if err != nil {
            return fmt.Errorf("failure applying network patch: %w", err)
        }

        outCfg, err := out.Config()
        if err != nil {
            return fmt.Errorf("failure converting result to bytes: %w", err)
        }

        retData.BootstrapData, err = outCfg.EncodeString(encoder.WithComments(encoder.CommentsDisabled))
        if err != nil {
            return fmt.Errorf("failure converting config to string: %w", err)
        }
    }

    return nil
}
```

### CRD Updates

The CRD definitions in `config/crd/bases/bootstrap.cluster.x-k8s.io_talosconfigs.yaml` and `config/crd/bases/bootstrap.cluster.x-k8s.io_talosconfigtemplates.yaml` need to be updated to include the new `ipAddress` field. This will be done by running `make generate` and `make manifests`.

## User Experience

### Example Usage

```yaml
spec:
  generateType: controlplane
  talosVersion: v1.10
  ipAddress:
    source: MachineStatus
    addressType: InternalIP
    interface: eth0
    subnetPrefix: /16
    defaultGateway: 172.16.0.1
```

### Documentation Updates

The README.md file will be updated to include a new section:

```markdown
### Dynamic IP Address Configuration

CABPT supports configuring network interfaces with IP addresses dynamically allocated by the IPAM provider:

spec:
  generateType: controlplane
  talosVersion: v1.10
  ipAddress:
    source: MachineStatus
    addressType: InternalIP
    interface: eth0
    subnetPrefix: /16
    defaultGateway: 172.16.0.1

This configuration will:
1. Wait for the IPAM provider to allocate an IP address to the Machine
2. Fetch the address from Machine.status.addresses with type "InternalIP"
3. Configure network interface "eth0" with the allocated IP and specified subnet
4. Set the default gateway to 172.16.0.1
```

## Testing Strategy

### Unit Tests

1. Test `hasMachineIPAllocated` function with various scenarios:
   - Machine with IP address allocated
   - Machine without IP address
   - Different address types
   - Machine pool (should always return true)

2. Test reconcile function to ensure it requeues when IP is not allocated.

3. Test network configuration generation with various inputs.

### Integration Tests

1. Test full workflow with a mock IPAM provider:
   - Create a TalosConfig with MachineStatus IPAddress source
   - Create a Machine without an IP
   - Verify the controller requeues
   - Add an IP to the Machine
   - Verify the controller proceeds and generates correct config

2. Test with different network configurations (subnet, gateway, etc.)

### End-to-End Tests

1. Full end-to-end test with the IPAM InCluster provider in a test cluster.

## Implementation Timeline

1. **Phase 1: API Changes** (1 week)
   - Add new types and fields
   - Update CRDs
   - Update API validation
   - Create unit tests for the new types

2. **Phase 2: Controller Logic** (2 weeks)
   - Implement IP allocation check
   - Update reconcile loop to wait for IP
   - Implement network configuration generation
   - Create unit tests for the updated controller

3. **Phase 3: Testing and Documentation** (1 week)
   - Create integration tests
   - Update project documentation
   - Create examples

4. **Phase 4: Review and Refinement** (1 week)
   - Address review feedback
   - Final testing
   - Prepare for release

## Risks and Mitigations

### Risks

1. **Backward Compatibility**: Changes might affect existing users who have created workarounds.
   - **Mitigation**: Ensure all changes are opt-in and don't affect existing behavior.

2. **Testing Complexity**: Testing with real IPAM providers adds complexity.
   - **Mitigation**: Create mock implementations for unit and integration tests.

3. **Performance**: Waiting for IP allocation might delay machine bootstrapping.
   - **Mitigation**: Use appropriate requeue intervals and document the behavior.

## Alternatives Considered

1. **Go Template Support in TalosConfig**: Implement full Go templating in the Talos machine configuration.
   - **Rejected**: This would be a more complex change with broader implications.

2. **Custom IPAM Controller**: Create a specialized controller just for IP allocation.
   - **Rejected**: The IPAM InCluster provider already exists and is well-integrated with CAPI.

3. **Webhook-Based Solution**: Use a mutating webhook to inject IP addresses.
   - **Rejected**: This would add unnecessary complexity to the architecture.

## Conclusion

The proposed implementation provides a straightforward solution to the current limitation where users need to hardcode IP addresses in their templates. By leveraging the existing IPAM InCluster provider and the standard CAPI Machine status fields, we can deliver this feature with minimal changes to the codebase while providing significant value to users.

This approach aligns with the project's design principles and leverages existing mechanisms like strategic patches to modify the machine configuration. It provides a robust solution that will work well with the existing ecosystem of CAPI providers.