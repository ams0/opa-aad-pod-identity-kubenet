# Prevent ARP spoofing on kubenet AKS clsuters

AAD Pod Identity in AKS is considered insecure when running on kubenet clusters ([reference](https://azure.github.io/aad-pod-identity/docs/configure/aad_pod_identity_on_kubenet/)). To prevent ARP spoofing from non-autorized pods, you can use a PodSecurityPolicy, but that's on the way to deprecation; the common way to implement policies now is with OpenPolicyAgent/Gatekeeper. Here's a quick guide on how to prevent pods to have the `NET_RAW' capability using constraints.

Install Gatekeeper:

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.3.0-beta.2/deploy/gatekeeper.yaml
```

Install the ConstraintTemplate and the constraint of type `K8sPSPCapabilities`

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/capabilities/template.yaml
kubectl apply -f constraint.yaml
```

Confirm it works: first apply a pod with explicit NET_RAW capability and see it being refused:

```bash
kubectl apply -f pod_cap_net_raw-disallowed.yaml

Error from server ([denied by capabilities-demo] container <alpine> is not dropping all required capabilities. Container must drop all of ["NET_RAW"]
[denied by capabilities-demo] container <alpine> has a disallowed capability. Allowed capabilities are NONE): error when creating "pod_cap_net_raw-disallowed.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [denied by capabilities-demo] container <alpine> is not dropping all required capabilities. Container must drop all of ["NET_RAW"]
[denied by capabilities-demo] container <alpine> has a disallowed capability. Allowed capabilities are NONE
```

Even if you don't specify the capability explicitly, the request is denied:

```bash
kubectl apply -f pod_cap_net_raw-none.yaml

Error from server ([denied by capabilities-demo] container <alpine> is not dropping all required capabilities. Container must drop all of ["NET_RAW"]): error when creating "pod_cap_net_raw-none.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [denied by capabilities-demo] container <alpine> is not dropping all required capabilities. Container must drop all of ["NET_RAW"]
```

Finally, a pod that explicitly drops the capability is allowed:

```bash
kubectl apply -f pod_cap_net_raw-allowed.yaml
pod/opa-disallowed created
```

If you want to check which capabilites are enabled, you can use the `capsh` command:

```bash
kubectl exec opa-disallowed -- apk add -U libcap; capsh --print
```

Install the AAD Pod Identity with Helm:

```bash
helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts

helm install --create-namespace -n aad-pod-identity --set nmi.allowNetworkPluginKubenet=true aad-pod-identity aad-pod-identity/aad-pod-identity
```
