# test-crossplane

## kind

```
go install sigs.k8s.io/kind@v0.23.0

kind create cluster

kubectl cluster-info --context kind-kind

kubectl get node
kubectl get pod -A

kind delete cluster

```

## crossplane

```
➜  test-crossplane git:(main) ✗ kind --version
kind version 0.23.0
➜  test-crossplane git:(main) ✗ helm version
version.BuildInfo{Version:"v3.15.2", GitCommit:"1a500d5625419a524fdae4b33de351cc4f58ec35", GitTreeState:"clean", GoVersion:"go1.22.4"}

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane 

kubectl get pods -n crossplane-system
kubectl get deployments -n crossplane-system

curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" | sh
sudo mv crossplane /usr/local/bin
crossplane --help

➜  test-crossplane git:(main) ✗ crossplane version
Client Version: v1.17.0
Server Version: v1.17.0

```

```
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-cloudformation
spec:
  package: xpkg.upbound.io/upbound/provider-aws-cloudformation:v1.12.0
EOF

kubectl get providers
```

```
sed -e 's/abcdcloud-peak-res-cprnd-dev:abcd-az-residentAdmins-dev/default/' -e 's/; Managed by samlaz//' ~/.aws/credentials > ./aws_credentials.txt
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws_credentials.txt
rm ./aws_credentials.txt
```

```
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-provider-config
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF
```

```
cat <<EOF | kubectl apply -f -
apiVersion: cloudformation.aws.upbound.io/v1beta1
kind: Stack
metadata:
  annotations:
    meta.upbound.io/example-id: cloudformation/v1beta1/stack
    uptest.upbound.io/update-parameter: '{"tags":{"update-test-tag":"val"}}'
  labels:
    testing.upbound.io/example-name: keyspaces
  name: crossplane-keyspace-gary
spec:
  providerConfigRef:
    name: aws-provider-config
  forProvider:
    name: keyspaces-stack
    # parameters:
    #   KeyspaceName: "test_keyspace"
    region: us-west-2
    templateBody: >
      {
        "AWSTemplateFormatVersion": "2010-09-09",
        "Resources": {
          "MultiRegionKeyspace": {
            "Type": "AWS::Cassandra::Keyspace",
            "Properties": {
              "KeyspaceName": "test_keyspace_gary",
              "ReplicationSpecification": {
                "ReplicationStrategy": "MULTI_REGION",
                "RegionList": ["us-west-2", "us-east-2"]
              }
            }
          }
        }
      }
EOF
```

```
kubectl delete stacks crossplane-keyspace-gary
```