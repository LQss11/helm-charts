# Operator EKS Add-on

This is a wrapper chart for installing EKS add-on. Charts required for the add-on are added as a dependency to this chart. Chart itself doesn't contain any templates or configurable properties.

## Version Mapping
| `operator-addon-chart` | `datadog-operator` | `datadog-crds` | Operator | Agent | Cluster Agent |
| - | - | - | - | - | - |
| 0.1.x | 1.0.5 | 1.0.1 | 1.0.3 | 7.43.1 | 7.43.1 | 

## Pushing Add-on Chart
Below steps have been validated using `Helm v3.12.0`.

* Build dependencies - this step is necessary to update the dependent charts under `charts/operator-eks-addon/charts` before we package it.

```sh
helm dependency build charts/operator-eks-addon
```

* Package chart - this step creates a chart archive e.g. `operator-eks-addon-0.1.0.tgz`
```sh
helm package ./charts/operator-eks-addon
```

* Push chart to EKS repo - first we need to authenticate Helm with the repo, then we push the chart archive to the Marketplace repository. This will upload the chart at `datadog/helm-charts/operator-eks-addon` and tag it with version `0.1.0`. See [ECR documentation][eks-helm-push] for more details.
```sh
aws-vault exec prod-engineering -- aws ecr get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com
helm push operator-eks-addon-0.1.0.tgz oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/datadog/helm-charts
```

* Validation - with AWS CLI we can list images in a give reposotiry.
```sh
aws-vault exec prod-engineering -- aws ecr describe-images --registry-id 709825985650 --region us-east-1  --repository-name datadog/helm-charts/operator-eks-addon
{
    "imageDetails": [
        {
            "registryId": "709825985650",
            "repositoryName": "datadog/helm-charts/operator-eks-addon",
            "imageDigest": "sha256:d6e54cbe69bfb962f0a4e16c9b29a7572f6aaf479de347f91bea8331a1a867f9",
            "imageTags": [
                "0.1.0"
            ],
            "imageSizeInBytes": 63269,
            "imagePushedAt": 1690215560.0,
            "imageManifestMediaType": "application/vnd.oci.image.manifest.v1+json",
            "artifactMediaType": "application/vnd.cncf.helm.config.v1+json"
        }
    ]
}
```

## Pushing Container Images
Images required during add-on instasllation must be available through the EKS marketplace repository. Each image can be copied simply by `crane copy`. Make sure all referenced tags are uploaded to the respective repository.
```sh
aws-vault exec prod-engineering -- aws ecr get-login-password --region us-east-1|crane auth login --username AWS --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com

❯ crane copy gcr.io/datadoghq/operator:1.0.3 709825985650.dkr.ecr.us-east-1.amazonaws.com/datadog/operator:1.0.3
```

To validate, describe the repository
```sh
aws-vault exec prod-engineering -- aws ecr describe-images --registry-id 709825985650 --region us-east-1  --repository-name datadog/operator
..
        {
            "registryId": "709825985650",
            "repositoryName": "datadog/operator",
            "imageDigest": "sha256:e7ad530ca73db7324186249239dec25556b4d60d85fa9ba0374dd2d0468795b3",
            "imageTags": [
                "1.0.3"
            ],
..
```

[eks-helm-push]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html