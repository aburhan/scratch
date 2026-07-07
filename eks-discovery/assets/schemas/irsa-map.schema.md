# irsa-map.tsv schema

Tab-separated, no header. One row per IRSA binding across all clusters.

## Columns

| #   | Name            | Description                    |
| --- | --------------- | ------------------------------ |
| 1   | cluster         | EKS cluster name               |
| 2   | namespace       | Kubernetes namespace           |
| 3   | service_account | Kubernetes ServiceAccount name |
| 4   | role_arn        | IAM role ARN                   |
| 5   | mechanism       | `irsa`                         |

## Example

```text
prod-east-1 storefront checkout-sa arn:aws:iam::123456789012:role/checkout irsa
prod-east-1 payments api-sa arn:aws:iam::123456789012:role/payments-api irsa
```
