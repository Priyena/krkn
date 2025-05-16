# Application Outage Scenario

The Application Outage scenario plugin enables simulating network connectivity issues for specific pods by creating temporary NetworkPolicies that block traffic.

## Overview

This scenario creates a network outage for selected pods by applying a NetworkPolicy that blocks ingress and/or egress traffic. After the specified duration, the NetworkPolicy is automatically removed, restoring normal network connectivity.

## Use Cases

- Test application resilience to network disruptions
- Validate service mesh retry and timeout configurations
- Simulate temporary network partitions between components
- Test failover mechanisms when connectivity is lost

## Configuration Parameters

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| namespace | string | Namespace where the NetworkPolicy will be applied | (Required) |
| pod_selector | dict | Pod selector labels for the NetworkPolicy (matchLabels) | (Required) |
| block | list | Traffic types to block (`Ingress`, `Egress`, or both) | [Ingress, Egress] |
| duration | int | Duration in seconds to keep traffic blocked | 60 |

> **Important**: The `pod_selector` parameter must be a non-empty dictionary of Kubernetes labels to select target pods. 
> A NetworkPolicy with an empty pod selector would apply to all pods in the namespace, which is not supported by this plugin.

## Sample Scenario File

```yaml
application_outage:
  namespace: "app-namespace"
  pod_selector:
    app: backend
    tier: database
  block:
    - Ingress  # Block only incoming traffic
  duration: 120  # Block for 2 minutes
```

## How to Run

```bash
# Run the scenario with a sample file
python run_kraken.py --config config/config.yaml

# Before running, ensure the scenario is included in config.yaml under chaos_scenarios:
# - application_outages_scenarios:
#     - /krkn/scenarios/openshift/app_outage.yaml
```

## How It Works

1. The plugin creates a NetworkPolicy in the specified namespace with the given pod selector
2. The NetworkPolicy is configured with empty ingress/egress rules (which explicitly denies all traffic)
3. The policy targets only pods matching the selector labels
4. After the specified duration, the NetworkPolicy is deleted, restoring normal traffic

## Metrics and Observability

During the test, you can observe:
- Network connectivity issues between components
- Application health metrics and logs
- Retry patterns and timeouts
- Failover behavior if applicable

## CI Testing

The application outage scenario can be tested using the CI test script:

```bash
# Run the application outage test
./CI/tests/test_app_outages.sh
```

This script will:
1. Set up a test namespace
2. Create a test application
3. Apply network policy to block traffic
4. Verify the traffic blocking works correctly
5. Clean up resources after testing

## Limitations

- The NetworkPolicy applies to all pods matching the selector in the specified namespace
- For more complex traffic manipulation, consider using other approaches like service mesh rules
- Empty pod selectors are not supported - you must specify at least one label

## Troubleshooting

- Verify that the NetworkPolicy is created and deleted as expected with:
  ```
  kubectl get networkpolicy -n <namespace>
  ```
- Check application logs for connectivity issues during the test period
- Ensure the pod selector matches the intended pods
- If you see errors about empty pod selectors, make sure to specify at least one label in your pod_selector

## Manual Recovery

In case the scenario exits prematurely without cleaning up, you can manually delete the NetworkPolicy:

```bash
kubectl delete networkpolicy/krkn-deny-* -n <targeted-namespace>
```

## Testing the Plugin

### Unit Tests

To run the unit tests for the application outage plugin:

```bash
# Run only the application outage tests
python -m unittest tests/application_outage_test.py

# Run all tests with coverage
python -m coverage run -m unittest discover -s tests/
python -m coverage report -m
```

### Integration Tests

For testing with Kubernetes:

1. Set up a test configuration file:

```yaml
# Save this as test-config.yaml in your Kraken root directory
kraken:
  chaos_scenarios:
    - application_outages_scenarios:
        - /krkn/test_app_outage.yaml  # Full path to your scenario file
  kubeconfig_path: ~/.kube/config

tunings:
  wait_duration: 10
  iterations: 1
  daemon_mode: False

cerberus:
  cerberus_enabled: False
```

2. Create a test scenario:

```yaml
# Save this as test_app_outage.yaml in your Kraken root directory
application_outage:
  namespace: "default"                # The namespace containing your application pods
  pod_selector:                       # Target specific pods with these labels
    app: your-app-name                # Required: at least one label must be specified
    component: your-component         # Optional: add more labels for more specific targeting
  block:
    - Ingress                         # Block incoming traffic
    # - Egress                        # Uncomment to also block outgoing traffic
  duration: 30                        # Duration in seconds before traffic is restored
```

3. Run the integration test:

```bash
# Run the integration test directly
python -m tests.application_outage_integration_test

# Or run through Kraken
python run_kraken.py --config test-config.yaml
```

### Manual Testing

To test the network policy creation and traffic blocking:

1. Create a test deployment:
```bash
kubectl create deployment nginx-test --image=nginx
kubectl label deployment nginx-test app=nginx test-selector=true
kubectl expose deployment nginx-test --port=80 --name=nginx-service
```

2. Verify initial connectivity:
```bash
kubectl run -i --rm test-pod --image=busybox --restart=Never -- wget -O- nginx-service
```

3. Run the application outage scenario
4. Verify connectivity is blocked during the outage
5. After the duration expires, verify connectivity is restored:
```bash
kubectl run -i --rm test-pod --image=busybox --restart=Never -- wget -O- nginx-service
```


# Git Commands for Application Outage Fix

Here are the commands to create a branch, commit your changes, and push them:

## Create a New Branch

```bash
# Create a new branch from the current state
git checkout -b fix-application-outage-empty-pod-selector main
```

## Check Modified Files

```bash
# View what files have been modified
git status
```

## Add Files to Staging

```bash
# Add the modified plugin file
git add krkn/scenario_plugins/application_outage/application_outage_scenario_plugin.py

# Add the test files
git add tests/application_outage_test.py
git add tests/application_outage_integration_test.py

# Add documentation updates
git add docs/application_outages.md

# Add example files
git add test_app_outage.yaml test-config.yaml
```

## Create a Commit with a Good Message

```bash
# Commit with a detailed message
git commit -s -m "Fix application outage plugin to reject empty pod selectors

This change:
- Adds validation to ensure pod_selector is not empty
- Improves error messaging for better debugging
- Updates NetworkPolicy template with proper labels
- Makes cerberus import more robust for testing
- Updates documentation with clear examples
- Adds comprehensive tests for validation

Fixes issue where empty pod selectors caused NetworkPolicy creation to fail
with 'kubectl create' returning non-zero exit status."
```

## Push the Branch to Your Fork

```bash
# Push the branch to your fork (assuming you have a fork set up)
git push -u origin fix-application-outage-empty-pod-selector
```

After pushing, you can create a pull request through the GitHub interface to contribute your fix back to the main repository.