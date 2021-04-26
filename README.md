# terraform-aws-elasticsearch-cloudwatch-sns-alarms

[![Build Status](https://travis-ci.com/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms.svg?branch=master)](https://travis-ci.org/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms)
[![Latest Release](https://img.shields.io/github/release/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms.svg)](https://github.com/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms/releases)

Terraform module that configures the [recommended Amazon ElasticSearch Alarms](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/cloudwatch-alarms.html) using CloudWatch and sends alerts to an SNS topic.  By default, this module creates an SNS topic, but it can be configured to point to an existing SNS topic (see [example](./examples/use-existing-sns/main.tf))

`v1.x` supports terraform `v0.12` syntax!

This project is inspired by [CloudPosse](https://github.com/cloudposse)

It's 100% Open Source and licensed under the [APACHE2](LICENSE).

## Metrics and Alarms

| area       | metric                    | operator | threshold | rationale                                                                                                                              |
|------------|---------------------------|----------|-----------|----------------------------------------------------------------------------------------------------------------------------------------|
| Sharding   | ClusterStatus.red         | `>=`     | 1         | At least one primary shard and its replicas are not allocated to a node                                                                |
| Sharding   | ClusterStatus.yellow      | `>=`     | 1         | At least one replica shard is not allocated to a node                                                                                  |
| Storage    | FreeStorageSpace          | `<=`     | 20480 MB  | A node in your cluster is down to low storage space.                                                                                   |
| Storage    | ClusterIndexWritesBlocked | `>=`     | 1         | Your cluster is blocking write requests.                                                                                               |
| Node Count | Nodes                     | `<`      | `x`       | This alarm indicates that at least one node in your cluster has been unreachable for one day                                           |
| Snapshot   | AutomatedSnapshotFailure  | `>=`     | 1         | An automated snapshot failed. This failure is often the result of a red cluster health status.                                         |
| CPU        | CPUUtilization            | `>=`     | 80 %      | 100% CPU utilization isn't uncommon, but sustained high usage is problematic. Consider using larger instance types or more instances.  |
| Memory     | JVMMemoryPressure         | `>=`     | 80 %      | The cluster could encounter out of memory errors if usage increases. Consider scaling vertically.                                      |
| CPU        | MasterCPUUtilization      | `>=`     | 80 %      | Consider using larger instance types for your dedicated master nodes.                                                                  |
| Memory     | MasterJVMMemoryPressure   | `>=`     | 80 %      | Consider using larger instance types for your dedicated master nodes.                                                                  |
| KMS        | KMSKeyError               | `>=`     | 1         | The KMS encryption key that is used to encrypt data at rest in your domain is disabled. Re-enable it to restore normal operations      |
| Memory     | KMSKeyInaccessible        | `>=`     | 80 %      | The KMS encryption key that is used to encrypt data at rest in your domain has been deleted or has revoked its grants to Amazon ES     |

For more information please see: [recommended Amazon ElasticSearch Alarms](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/cloudwatch-alarms.html).

## Examples

See the [`examples/`](examples/) directory for working examples.

```hcl
resource "aws_elasticsearch_domain" "es" {
  domain_name           = "example"
  elasticsearch_version = "6.3"

  cluster_config {
    instance_type = "r4.large.elasticsearch"
  }

  snapshot_options {
    automated_snapshot_start_hour = 23
  }

  tags = {
    Domain = "TestDomain"
  }
}

module "es_alarms" {
  source         = "github::https://github.com/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms.git?ref=master"
  domain_name    = "example"
  tags = {
    Domain = "TestDomain"
  }
}
```

You can alternatively have this module not create an SNS incase you have existing ones created elsewhere.

```hcl
module "es_alarms" {
  source           = "github::https://github.com/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms.git?ref=master"
  domain_name      = "example"
  sns_topic        = "arn:aws:sns:us-east-1:123456123456:sns-to-slack"   # < Put your full SNS ARN here, if necessary read from var or a resource
  create_sns_topic = false
  tags = {
    Domain = "TestDomain"
  }
}
```


## Inputs

| Name                                          | Description | Type | Default | Required |
|-----------------------------------------------|-------------|:----:|:-------:|:--------:|
| `domain_name`                                 | The Elasticserach domain name you want to monitor. | string | - | yes |
| `alarm_cluster_status_is_yellow_periods`      | The number of periods before triggering the cluster status is yellow, raise this if desired to make less noisy | number | `1` | no |
| `alarm_free_storage_space_too_low_periods`    | The number of periods before triggering the disk space is low, raise this if desired to make less noisy | number | `1` | no |
| `alarm_name_postfix`                          | Alarm name postfix | string | `""` | no |
| `alarm_name_prefix`                           | Alarm name prefix | string | `""` | no |
| `cpu_utilization_threshold`                   | The maximum percentage of CPU utilization | string | `80` | no |
| `free_storage_space_threshold`                | The minimum amount of available storage space in MiB. | string | `20480` | no |
| `jvm_memory_pressure_threshold`               | The maximum percentage of the Java heap used for all data nodes in the cluster | string | `80` | no |
| `master_cpu_utilization_threshold`            | The maximum percentage of CPU utilization of master nodes | string | `""` | no |
| `master_jvm_memory_pressure_threshold`        | The maximum percentage of the Java heap used for master nodes in the cluster | string | `""` | no |
| `min_available_nodes`                         | The minimum available (reachable) nodes to have, set to non-zero to enable alarm | string | `0` | no |
| `monitor_automated_snapshot_failure`          | Enable monitoring of automated snapshot failure | bool | `true` | no |
| `monitor_cluster_index_writes_blocked`        | Enable monitoring of cluster index writes being blocked | bool | `true` | no |
| `monitor_cluster_status_is_red`               | Enable monitoring of cluster status is in red | bool | `true` | no |
| `monitor_cluster_status_is_yellow`            | Enable monitoring of cluster status is in yellow | bool | `true` | no |
| `monitor_cpu_utilization_too_high`            | Enable monitoring of CPU utilization is too high | bool | `true` | no |
| `monitor_free_storage_space_too_low`          | Enable monitoring of cluster average free storage is to low | bool | `true` | no |
| `monitor_jvm_memory_pressure_too_high`        | Enable monitoring of JVM memory pressure is too high | bool | `true` | no |
| `monitor_kms`                                 | Enable monitoring of KMS-related metrics, enable if using KMS | bool | `false` | no |
| `monitor_master_cpu_utilization_too_high`     | Enable monitoring of CPU utilization of master nodes are too high. Only enable this when dedicated master is enabled | bool | `false` | no |
| `monitor_master_jvm_memory_pressure_too_high` | Enable monitoring of JVM memory pressure of master nodes are too high. Only enable this wwhen dedicated master is enabled | bool | `false` | no |
| `create_sns_topic`                            | Will create an SNS topic, if you set this to false you MUST set `sns_topic` to a FULL ARN | bool | `true` | no |
| `sns_topic`                                   | SNS topic you want to specify. If leave empty, it will use a prefix and a timestamp appended.  If `create_sns_topic` is set to false, this MUST be a FULL ARN | string | `""` | no |
| `sns_topic_postfix`                           | SNS topic postfix | string | `""` | no |
| `sns_topic_prefix`                            | SNS topic prefix | string | `""` | no |
| `tags`                                        | Tags to associate with all created resources | map | `{}` | no |

## Outputs

| Name | Description |
|------|-------------|
| `sns_topic_arn`  | The ARN of the SNS topic |
| `sns_topic_name` | The SNS topic name |

## Share the Love

Like this project? Please give it a ★ on [our GitHub](https://github.com/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms)!

## Help

**Got a question?**

File a GitHub [issue](https://github.com/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms/issues).

### Bug Reports & Feature Requests

Please use the [issue tracker](https://github.com/dubiety/terraform-aws-elasticsearch-cloudwatch-sns-alarms/issues) to report any bugs or file feature requests.

## License

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

See [LICENSE](LICENSE) for full details.

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
