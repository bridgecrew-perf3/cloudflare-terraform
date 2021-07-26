# Terraform existing Cloudflare infrastructure

## Requirements

- A Cloudflare account with resources defined (e.g. a few zones, some load balancers, spectrum applications, etc)
- A valid Cloudflare API key and sufficient permissions to access the resources you are requesting via the API
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

# warning!!
# status = "active" 
# this statement cause issues, just delete all rows from the cloudflare_zone.tf file

# this will output terraform import commands to copy/paste and execute
cf-terraforming -c .cf-terraforming.yaml import --resource-type "cloudflare_zone"

# extract zones ids
cat cloudflare_zone.tf  | grep resource | awk '{print $3}' | sed s/\"terraform_managed_resource_//g | sed s/\"//g >> zones.txt

# extract zone records for all zones and save to cloudflare_record.tf file
for i in $(cat zones.txt | xargs); do cf-terraforming -c .cf-terraforming.yaml generate --resource-type "cloudflare_record" -z "$i" >> cloudflare_record.tf; done

# this will output terraform import commands to copy/paste and execute
for i in $(cat zones.txt | xargs); do cf-terraforming -c .cf-terraforming.yaml import --resource-type "cloudflare_record" -z "$i"; done

# yet another example: extract page rules for a specific zone (by ID) and safe to cloudflare_page_rule.tf file
cf-terraforming -c .cf-terraforming.yaml import --resource-type "cloudflare_page_rule" -z "$CLOUDFLARE_ZONE_ID" >> cloudflare_page_rule.tf

terraform plan
terraform apply
```

4. (OPTIONAL) Add new firewall rule

```
# where $i is zone id

resource "cloudflare_filter" "wordpress" {
  zone_id = "$i"
  description = "Wordpress break-in attempts that are outside of the office"
  expression = "(http.request.uri.path ~ \".*wp-login.php\" or http.request.uri.path ~ \".*xmlrpc.php\") and ip.src ne 192.0.2.1"
}

resource "cloudflare_firewall_rule" "wordpress" {
  zone_id = "$i"
  description = "Block wordpress break-in attempts"
  filter_id = cloudflare_filter.wordpress.id
  action = "block"
}
```

5. (OPTIONAL) Migrate to Terraform cloud

- create file **remote.tf**

```
# file remote.tf

terraform {
  backend "remote" {
    organization = "Your_Terraform_Cloud_Organization"

    workspaces {
      name = "Your_Terraform_Cloud_Workspace"
    }
  }
}
```

- remotely execute Terraform

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
