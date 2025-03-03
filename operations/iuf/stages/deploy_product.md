# `deploy-product`

The `deploy-product` stage uses Loftsman to deploy product microservices to the system. The microservices are specified in the `loftsman` entry in each product's `iuf-product-manifest.yaml` file within the product distribution file.

`deploy-product` details are explained in the following sections:

- [Impact](#impact)
- [Input](#input)
- [Execution details](#execution-details)
- [Example](#example)

## Impact

The `deploy-product` stage changes the running state of the system.

## Input

The following arguments are most often used with the `deploy-product` stage. See `iuf -h` and `iuf run -h` for additional arguments.

| Input                                  | `iuf` Argument              | Description                                                                  |
|----------------------------------------|-----------------------------|------------------------------------------------------------------------------|
| Activity                               | `-a ACTIVITY`               | Activity created for the install or upgrade operations                       |
| Site variables                         | `-sv SITE_VARS`             | Path to YAML file containing site defaults and any overrides                 |
| Recipe variables                       | `-rv RECIPE_VARS`           | Path to YAML file containing recipe variables provided by HPE                |
| `sat bootprep` configuration directory | `-bpcd BOOTPREP_CONFIG_DIR` | Directory containing `sat bootprep` configuration files and recipe variables |

## Execution details

The code executed by this stage exists within IUF. See the `deploy-product` entry in `/usr/share/doc/csm/workflows/iuf/stages.yaml`
and the corresponding files in `/usr/share/doc/csm/workflows/iuf/operations/` for details on the commands executed.

## Example

(`ncn-m001#`) Execute the `deploy-product` stage for activity `admin-230127` using the `/etc/cray/upgrade/csm/admin/site_vars.yaml` file and the `product_vars.yaml` file found in the `/etc/cray/upgrade/csm/admin` directory.

```bash
iuf -a admin-230127 run -sv /etc/cray/upgrade/csm/admin/site_vars.yaml -bpcd /etc/cray/upgrade/csm/admin -r deploy-product
```
