---
id: ea918976-368e-462c-9d0f-a025df7ee5ff
binder: binder-mvvle32a-1
folder: folder-mvw0u6xp
---
DevOps | devplus-platform/docs/COST_OVERRIDES.md

Source: DevPlusInc/devplus-platform/docs/COST_OVERRIDES.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/COST_OVERRIDES.md).

# Cost Overrides (`devplus.hcl`)

DevPlus uses Infracost to estimate and calculate deployment costs. If you have resources that Infracost can’t price correctly (or you want to provide a custom fixed cost), you can add an optional `devplus.hcl` file to your module repo to override per-resource costs. (We use `devplus.hcl` so we can add more module-level optimizations/config later.) The InfraSync-era name `infrasync.hcl` is no longer read: rename it.

## Where to put `devplus.hcl`

- Place `devplus.hcl` at the root of your module (next to your `*.tf` files; for a monorepo module, in its subdirectory).
- When the module is deployed, the executor reads this file from the downloaded module (the wrapper's `catalog_item` module) and applies the overrides to both:
  - **Estimated cost** (from the plan)
  - **Actual cost** (from state after apply)
- A `devplus.hcl` in the generated wrapper directory is also read, after the module's, so its rules win. That is an operator escape hatch, not something a module author ships.
- Overrides only adjust an Infracost result: when the executor has no `INFRACOST_API_KEY`, cost estimation is skipped and no overrides run.

## File format

`devplus.hcl` contains one or more `resource_override` blocks:

```hcl
resource_override {
  # Matchers (at least one is required)
  address       = "aws_instance.web"                 # exact or suffix match (recommended)
  # address_regex = "^module\\.catalog_item\\..*web" # optional RE2 regex
  # type         = "aws_instance"
  # name         = "web"

  # Override values (set one or both)
  monthly_cost = 10.00
  hourly_cost  = 0.02

  # Optional condition (HCL boolean expression)
  # when = var.number_of_nodes > 3

  # Optional metadata
  reason = "Reserved instance pricing"
}
```

## Conditional + formula pricing

You can make overrides depend on the module inputs that were used for the deployment (e.g., number of nodes, CPU, memory).

### Conditional fixed pricing

```hcl
resource_override {
  address      = "null_resource.eks_cluster"
  when         = var.number_of_nodes <= 3
  monthly_cost = 200.00
  reason       = "Small cluster"
}

resource_override {
  address      = "null_resource.eks_cluster"
  when         = var.number_of_nodes > 3
  monthly_cost = 350.00
  reason       = "Large cluster"
}
```

### Formula pricing

Use `monthly_cost_expr` / `hourly_cost_expr` for HCL number expressions:

```hcl
resource_override {
  address           = "null_resource.eks_cluster"
  monthly_cost_expr = 73 + var.number_of_nodes * 30
  reason            = "Base fee + per-node pricing"
}
```

### Expression context

- `var.<name>`: module input values for the deployment (includes explicit inputs from the generated wrapper `main.tf` and any static defaults from the module’s `variable` blocks; after apply, missing values may be filled from the `module_outputs` output when available)
- `resource.address`, `resource.type`, `resource.name`: the matched resource identity

Supported helper functions: `tonumber`, `tostring`, `tobool`, `length`, `ceil`, `floor`, `round`, `min`, `max`, `abs`.

## Matching rules

- `address` matches either the full Infracost address or the **suffix** of it, so you can usually write addresses as they appear inside your module (e.g. `aws_instance.web`) without needing the wrapper prefix (e.g. `module.catalog_item.`).
- Indexed resources like `aws_instance.web[0]` are matched when you specify `address = "aws_instance.web"`. If you need more control, use `address_regex`.
- If multiple overrides match the same resource, they are applied **in file order** (later matches win).
- Overrides can price resources Infracost does not list at all (e.g. `null_resource`): a planned (non-delete) or state resource that matches an override is added to the estimate, then priced by it.
- Each overridden resource records what happened in its cost metadata under `devplus_cost_override` (original monthly/hourly cost plus every applied rule with its `source` file and `reason`); totals are recalculated.

## Notes / constraints

- A single override block cannot set both `monthly_cost` and `monthly_cost_expr` (same for `hourly_*`).
- Each block needs at least one matcher and at least one cost; fixed costs must be `>= 0`. A file that fails to parse or validate (including an invalid `address_regex`) is rejected as a whole: the executor logs a warning and applies no overrides.
- If an expression can’t be evaluated (e.g., it references a missing `var.*`), DevPlus logs a warning and skips that override. An expression that evaluates to a negative number is skipped the same way.
