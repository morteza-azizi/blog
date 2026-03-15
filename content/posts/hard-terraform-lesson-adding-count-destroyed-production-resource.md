+++
date = '2025-03-15'
draft = false
title = 'A Hard Terraform Lesson: When Adding count Tried to Destroy My Production Resource'
tags = ['terraform', 'azure', 'infrastructure', 'devops', 'container-registry', 'cost-optimization']
+++

# A Hard Terraform Lesson: When Adding count Tried to Destroy My Production Resource

While working on our infrastructure, we made a cost optimization decision around Azure Container Registry (ACR).

Running a dedicated container registry for every environment did not make much sense. The cost was unnecessary, and lower environments could safely share one.

So we decided on the following approach:

- **DEV, ACC, and TEST** → use a shared container registry
- **Production** → keep a dedicated registry

From an architectural perspective, this was a simple and sensible improvement.

From a Terraform perspective, it turned into a tricky lesson.

## The Refactor

Originally, the Azure Container Registry was always created by Terraform.

```hcl
resource "azurerm_container_registry" "acr" {
  name                = "example-acr"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Basic"
}
```

To avoid creating the registry in the lower environments, I introduced a count condition.

```hcl
resource "azurerm_container_registry" "acr" {
  count = var.environment == "prod" ? 1 : 0

  name                = "example-acr"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Basic"
}
```

The intention was straightforward:

- **Production** → count = 1 → registry exists
- **DEV / ACC / TEST** → count = 0 → no registry created

The infrastructure design itself had not changed.

But when I ran:

```bash
terraform plan
```

Terraform wanted to destroy the existing registry and create a new one.

That was unexpected — and dangerous.

## Why Terraform Planned to Destroy the Resource

Terraform does not identify infrastructure primarily by cloud resource IDs.

Instead, Terraform tracks infrastructure through resource addresses that connect:

1. the Terraform configuration
2. the Terraform state
3. the real infrastructure in the cloud

Originally, the resource address was:

```
azurerm_container_registry.acr
```

After introducing count, the address became:

```
azurerm_container_registry.acr[0]
```

Even though there is still only one resource, Terraform now treats it as an indexed instance.

From Terraform's perspective:

- **OLD ADDRESS:** `azurerm_container_registry.acr`
- **NEW ADDRESS:** `azurerm_container_registry.acr[0]`

These are two different resources.

So Terraform assumed:

1. the old resource disappeared
2. a new resource should be created

Which resulted in a destroy and recreate plan.

## Why This Was a Serious Problem

In some cases, recreating infrastructure is acceptable.

But the container registry was critical infrastructure.

It contained container images used by multiple environments and deployment pipelines. Destroying and recreating it could have resulted in:

- loss of stored images
- broken CI/CD pipelines
- disruption across environments

This was not a risk I was willing to take.

## The Solution: Migrating the Terraform State

The fix was to tell Terraform that the existing resource simply moved to a new address.

This can be done using the `terraform state mv` command.

```bash
terraform state mv \
  azurerm_container_registry.acr \
  azurerm_container_registry.acr[0]
```

This command does not change the infrastructure.

It only updates the Terraform state so Terraform understands that the existing resource now belongs to the new address.

After migrating the state, running `terraform plan` again showed no destructive changes.

## A Better Alternative: moved Blocks

Starting with Terraform 1.1, there is a cleaner and safer way to handle this using a `moved` block.

```hcl
moved {
  from = azurerm_container_registry.acr
  to   = azurerm_container_registry.acr[0]
}
```

This approach makes the migration declarative and version-controlled, which is especially useful when working in teams.

## Lessons Learned

This experience reinforced several important Terraform lessons.

**Refactoring Terraform can change resource identity**

Even if the infrastructure stays the same, changing the configuration can change how Terraform identifies resources.

**count introduces indexed resources**

When you add count, Terraform converts the resource into a list:

```
resource.name → resource.name[0]
```

That change affects the resource address.

**Always review Terraform plans carefully**

Unexpected destroy operations during refactoring are often caused by address mismatches.

**Use state migration when refactoring**

If a resource address changes, migrate the state instead of recreating infrastructure.

**Always back up state before modifying it**

State operations are powerful and should be handled carefully.

## Final Thought

Terraform is extremely powerful, but it is also very literal.

A small configuration refactor can unintentionally change how Terraform identifies infrastructure resources. When that happens, Terraform may attempt to destroy and recreate resources that should remain untouched.

Understanding the relationship between configuration, state, and resource addresses is essential when evolving infrastructure that supports production systems.

Sometimes the hardest Terraform lessons are not about cloud services or providers — but about how Terraform thinks about infrastructure identity.
