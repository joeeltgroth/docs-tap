# Troubleshoot Services Toolkit

This topic explains how you can troubleshoot issues related to working with services on
Tanzu Application Platform (commonly known as TAP).

For the limitations of services on Tanzu Application Platform, see
[Services Toolkit limitations](../reference/known-limitations.hbs.md).

## <a id="debug-dynamic-provisioning"></a> Debug `ClassClaim` and provisioner-based `ClusterInstanceClass`

This section provides guidance on how to debug issues related to using `ClassClaim`
and provisioner-based `ClusterInstanceClass`.
The approach starts by inspecting a `ClassClaim` and tracing back through the chain of
resources that are created when fulfilling the `ClassClaim`.

### <a id="prereq"></a> Prerequisites

To follow the steps in this section, you must have kubectl access to the cluster.

### <a id="inspect-class-claim"></a> Step 1: Inspect the `ClassClaim`, `ClusterInstanceClass`, and `CompositeResourceDefinition`

1. Inspect the status of `ClassClaim` by running:

   ```console
   kubectl describe classclaim claim-name -n NAMESPACE
   ```

   Where `NAMESPACE` is your namespace.

   From the output, check the following:

   - Check the status conditions for information that can lead you to the cause of the issue.
   - Check `.spec.classRef.name` and record the value.

1. Inspect the status of the `ClusterInstanceClass` by running:

   ```console
   kubectl describe clusterinstanceclass CLASS-NAME
   ```

   Where `CLASS-NAME` is the value of `.spec.classRef.name` you retrieved in the previous step.

   From the output, check the following:

   - Check the status conditions for information that can lead you to the cause of the issue.
   - Check that the `Ready` condition has status `"True"`.
   - Check `.spec.provisioner.crossplane` and record the value.

1. Inspect the status of the `CompositeResourceDefinition` by running:

   ```console
   kubectl describe xrd XRD-NAME
   ```

   Where `XRD-NAME` is the value of `.spec.provisioner.crossplane` you retrieved in the previous step.

   From the output, check the following:

   - Check the status conditions for information that can lead you to the cause of the issue.
   - Check that the `Established` condition has status `"True"`.
   - Check events for any errors or warnings that can lead you to the cause of the issue.
   - If both the `ClusterInstanceClass` reports `Ready="True"` and the `CompositeResourceDefinition`
     reports `Established="True"`, move on to the next section.

### <a id="inspect-comp-resource"></a> Step 2: Inspect the Composite Resource, the Managed Resources and the underlying resources

1. Check `.status.provisionedResourceRef` by running:

   ```console
   kubectl describe classclaim claim-name -n NAMESPACE
   ```

   Where `NAMESPACE` is your namespace.

   From the output, check the following:

   - Check `.status.provisionedResourceRef`, and record the values of `kind`, `apiVersion`, and `name`.

1. Inspect the status of the Composite Resource by running:

   ```console
   kubectl describe KIND.API-GROUP NAME
   ```

   Where:

   - `KIND` is the value of `kind` you retrieved in the previous step.
   - `API-GROUP` is the value of `apiVersion` you retrieved in the previous step without the `/<version>` part.
   - `NAME` is the value of `name` you retrieved in the previous step.

   From the output, check the following:

   - Check the status conditions for information that can lead you to the cause of the issue.
   - Check that the `Synced` condition has status `"True"`. If it doesn't then there was an issue creating
   the Managed Resources from which this Composite Resource is composed. Refer to `.spec.resourceRefs`
   in the output and for each:
     - Use the values of `kind`, `apiVersion`, and `name` to inspect the status of the Managed Resource.
     - Check the status conditions for information that can lead you to the cause of the issue.
   - Check events for any errors or warnings that can lead you to the cause of the issue.
   - If all Managed Resources appear healthy, move on to the next section.

### <a id="inspect-log"></a> Step 3: Inspect the events log

Inspect the events log by running:

```console
kubectl get events -A
```

From the output, check the following:

- Check for any errors or warnings that can lead you to the cause of the issue.
- If there are no errors or warnings, move on to the next section.

### <a id="inspect-secret"></a> Step 4: Inspect the secret

1. Check `.status.resourceRef` by running:

   ```console
   kubectl get classclaim claim-name -n NAMESPACE -o yaml
   ```

   Where `NAMESPACE` is your namespace.

   From the output, check the following:

   - Check `.status.resourceRef` and record the values `kind`, `apiVersion`, `name`, and `namespace`

1. Inspect the claimed resource, which is likely a secret, by running:

   ```console
   kubectl get secret NAME -n NAMESPACE -o yaml
   ```

   Where:

   - `NAME` is the `name` you retrieved in the previous step.
   - `NAMESPACE` is the `namespace` you retrieved in the previous step.

   If the secret is there and has data, then something else must be causing the issue.

### <a id="contact-support"></a> Step 5: Contact support

If you have followed the steps in this section and are still unable to discover the cause of the issue,
contact VMware Support for further guidance and help to resolve the issue.

## <a id="compositeresourcedef"></a> Unexpected error if `additionalProperties` is `true` in a CompositeResourceDefinition

**Symptom:**

When creating a `CompositeResourceDefinition`, if you set `additionalProperties: true` in the
`openAPIV3Schema` section, an error occurs during the validation step of the creation of any
`ClassClaim` that refers to a class that refers to the `CompositeResourceDefinitions`.

The error appears as follows:

```console
json: cannot unmarshal bool into Go struct field JSONSchemaProps.AdditionalProperties of type apiextensions.JSONSchemaPropsOrBool
```

**Solution:**

Rather than setting `additionalProperties: true`, you can set `additionalProperties: {}`.
This has the same effect, but does not cause unexpected errors.

## <a id="claim-rbac"></a> Unexpected addmission error when creating `ClassClaims`

**Symptom:**

Users who previously were able to create `ClassClaim` now get an admission
error similar to:
```console
user 'alice@example.com' cannot 'claim' from clusterinstanceclass 'bigcorp-rabbitmq'
```

This is the case, even though those users have been granted the `claim`
permission on `ClusterInstanceClasses` throguh either:
- a `Role` and a `RoleBinding`
- a `ClusterRole` and a `RoleBinding`

**Solution:**

Users need to have the cluster-level `claim` permission, granted via
`ClusterRole` and `ClusterRuleBinding`. Namespace-scoped permissions are not
enough anymore, to guard against unexpected access to resources in other
namespaces.

This change came in with Services Toolkit v0.12.0 and thus with Tanzu
Application Platform v1.7.0, you can read more about that change
[here](../reference/api/rbac.hbs.md).
<!--
Services Toolkit v0.12.0 for Tanzu Application Platform v1.7.0
Services Toolkit v0.11.1 for Tanzu Application Platform v1.6.3
Services Toolkit v0.10.3 for Tanzu Application Platform v1.5.5
-->

To allow users to create `ClassClaims` again you need to
- move from granting `claim` on `clusterinstanceclasses` from a `Role` to a `ClusterRole`
- move from binding this permission to a user from a `RoleBinding` to a `ClusterRoleBinding`
