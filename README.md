# Terraform existing Cloudflare infrastructure

## Requirements

- [cf-terraforming => 0.3.0](https://github.com/cloudflare/cf-terraforming)
- [Terraform => 0.14.9](https://www.terraform.io/downloads.html)

## Setup

1. Create file **.cf-terraforming.yaml** with Cloudflare e-mail and API key (used by cf-terraforming)

```
email: "your-cloudflare@registered.email"
key: "your Cloudflare API key"
```

2. Create file **main.tf** with Cloudflare e-mail and API key (used by Terraform)

```
terraform {
  required_providers {
    cloudflare = {
      source = "cloudflare/cloudflare"
      version = "~> 2.0"
    }
  }
}

provider "cloudflare" { 
  email   = "your-cloudflare@registered.email"
  api_key = "your Cloudflare API key"
}

```

3. Import existing Cloudflare infrastructure

```
terraform init

# this will create resource blocks, we automatically save to cloudflare_zone.tf file
cf-terraforming -c .cf-terraforming.yaml generate --resource-type "cloudflare_zone" >> cloudflare_zone.tf

# this will output terraform import commands to copy/paste and execute
cf-terraforming -c .cf-terraforming.yaml import --resource-type "cloudflare_zone"

# extract zones ids
cat cloudflare_zone.tf  | grep resource | awk '{print $3}' | sed s/\"terraform_managed_resource_//g | sed s/\"//g >> zones.txt

# extract zone records for all zones and save to cloudflare_record.tf file
for i in $(cat zones.txt | xargs); do cf-terraforming -c .cf-terraforming.yaml generate --resource-type "cloudflare_record" -z "$i" >> cloudflare_record.tf; done

# this will output terraform import commands to copy/paste and execute
for i in $(cat zones.txt | xargs); do cf-terraforming -c .cf-terraforming.yaml import --resource-type "cloudflare_record" -z "$i"; done

# extract page rules for a specific zone (by ID) and safe to cloudflare_page_rule.tf file
cf-terraforming -c .cf-terraforming.yaml import --resource-type "cloudflare_page_rule" -z "$CLOUDFLARE_ZONE_ID" >> cloudflare_page_rule.tf

terraform plan
terraform apply
```

4. (OPTIONAL) Migrate to Terraform cloud

- create file **remote.tf** and execute Terraform remotely

```
terraform {
  backend "remote" {
    organization = "Your_Terraform_Cloud_Organization"

    workspaces {
      name = "Your_Terraform_Cloud_Workspace"
    }
  }
}
```

```
terraform login
terraform init
# you will be asked to migrate existing .terraform/terraform.tfstate file, answer yes
# remove local terraform.tfstate
rm .terraform/terraform.tfstate
terraform plan
terraform apply
```

![Terraform Cloudflare](https://raw.githubusercontent.com/fabriziosalmi/cloudflare-terraform/main/cloudflare-terraform.png)
