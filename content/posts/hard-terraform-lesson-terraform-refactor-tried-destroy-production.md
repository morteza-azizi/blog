+++
date = '2026-03-15'
draft = false
title = 'A Hard Terraform Lesson: When a Terraform Refactor Tried to Destroy Production'
tags = ['terraform', 'azure', 'infrastructure', 'devops', 'container-registry', 'cost-optimization']
+++

# A Hard Terraform Lesson: When a Terraform Refactor Tried to Destroy Production

While working on our infrastructure, I proposed a cost optimization around Azure Container Registry (ACR). The decision was a classic architectural trade-off: **cost vs. isolation**.

On one side: running a dedicated registry for every environment (DEV, ACC, TEST, Production) was unnecessary spend. Lower environments could safely share one. On the other: production needed isolation—its own registry, its own images, no risk of cross-environment bleed.

Most of our resources—storage, networking, compute—are deployed to all four environments. ACR was the exception. We chose a hybrid model:

- **DEV, ACC, and TEST** → share one registry (e.g. in ACC)
- **Production** → keep a dedicated registry

In the end: one prod-specific ACR, one shared non-prod ACR. The architecture was clear. The Terraform implementation, less so.

Infrastructure evolves. That means you must design for safe refactoring—and protect production before you change anything. This post is about both.

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

So how did I not end up destroying it? Because I had already added `lifecycle { prevent_destroy = true }` to critical resources like the ACR. Call it paranoia or foresight—either way, when Terraform proposed that destroy, the lifecycle block would have refused the apply. The resource was never at risk. I still had to fix the root cause, but the protection did its job.

## Why This Would Have Been a Serious Problem

In some cases, recreating infrastructure is acceptable. But the container registry was critical. It contained container images used by multiple environments and deployment pipelines. Destroying and recreating it could have meant:

- loss of stored images
- broken CI/CD pipelines
- disruption across environments

I was protected—because the lifecycle block would have refused the apply. Terraform could still propose the destroy in the plan, but the resource could not actually be removed. That is exactly why I had added it in the first place. But what caused Terraform to propose the destroy?

## Why Terraform Planned to Destroy the Resource

Terraform does not identify infrastructure primarily by cloud resource IDs. Instead, it tracks infrastructure through resource addresses that connect the configuration, state, and real cloud resources.

Originally, the resource address was:

```
azurerm_container_registry.acr
```

After introducing count, the address became:

```
azurerm_container_registry.acr[0]
```

Even though there is still only one resource, Terraform now treats it as an indexed instance. From Terraform's perspective, these are two different resources:

- **OLD ADDRESS:** `azurerm_container_registry.acr`
- **NEW ADDRESS:** `azurerm_container_registry.acr[0]`

{{< mermaid >}}
flowchart LR
    subgraph Before["Before refactor (aligned)"]
        direction TB
        C1["Config: acr"] --> S1["State: acr"]
        S1 --> A1["Azure ACR"]
    end
    subgraph After["After count - mismatch"]
        direction TB
        C2["Config: acr[0]"]
        S2["State: acr"]
        A2["Azure ACR"]
        C2 -.->|mismatch| S2
        S2 --> A2
    end
{{< /mermaid >}}

Config says acr[0], State has acr. Terraform: old address gone, new address needed — destroy + create. Terraform assumed the old resource disappeared and a new one should be created—hence the destroy and recreate plan.

## The Solution: Migrating the Terraform State

The fix was to tell Terraform that the existing resource simply moved to a new address.

This can be done using the `terraform state mv` command.

```bash
terraform state mv \
  azurerm_container_registry.acr \
  azurerm_container_registry.acr[0]
```

This command does not change the infrastructure. It only updates Terraform's understanding of which real resource belongs to which address.

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

## The Protection: lifecycle prevent_destroy

The ACR resource looked like this—including the block that would have refused the apply:

```hcl
resource "azurerm_container_registry" "acr" {
  count = var.environment == "prod" ? 1 : 0

  name                = "example-acr"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Basic"

  lifecycle {
    prevent_destroy = true
  }
}
```

For production infrastructure you never want to lose by accident, that habit pays off.

## Lessons Learned

This experience reinforced several important Terraform lessons—and one architectural one.

**Design for safe refactoring from the start**

Infrastructure evolves. If you add lifecycle protection and state-migration habits before you need them, refactors become less risky. Treat "protecting production" as part of the design, not an afterthought.

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

**Add lifecycle prevent_destroy to critical resources before you need it**

I had it on the ACR before this refactor. When the bad plan appeared, that block would have blocked apply. Foresight pays off.

## Final Thought

Terraform is extremely powerful, but it is also very literal. A small configuration refactor can unintentionally change how Terraform identifies infrastructure resources. When that happens, Terraform may attempt to destroy and recreate resources that should remain untouched.

Understanding the relationship between configuration, state, and resource addresses is essential when evolving infrastructure that supports production systems. So is a mindset: **design infra so it can be refactored safely**. That means lifecycle blocks on critical resources, state migration instead of blind apply, and treating IaC not just as "infrastructure as code", but as infrastructure that protects production by default.

Sometimes the hardest Terraform lessons are not about cloud services or providers—but about how Terraform thinks about infrastructure identity, and how you design for change.
