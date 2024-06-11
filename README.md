# [@cndi/cdktf](https://jsr.io/@cndi/cdktf)

## goals

- enable use of Terraform Modules in CNDI
- reduce the size of the CNDI CLI in bytes
- reduce the complexity of the CNDI CLI in lines of code
- reduce the distance between the CNDI toolchain and the rest of the cdktf
  ecosystem

## why now?

As we were going to build support for CNDI's keyless authentication, we found
the
[aws/eks CNDI deployment target](https://github.com/polyseam/cndi/blob/main/src/outputs/terraform/aws/AWSEKSStack.ts)
included a ton of boilerplate especially related to authorization, which is
quite sensitive.

If we can meaningfully reduce the footprint of our implementation, we can reduce
the surface area of the code which needs to be carefully and securely
maintained.

## background

[polyseam/cndi](https://github.com/polyseam/cndi) creates cloud infrastructure
using Terraform to create clusters, and GitOps Manifests to create the rest.

The [cndi](https://github.com/polyseam/cndi) CLI does this by importing every
required Typescript API for each
[Terraform Provider](https://registry.terraform.io/browse/providers) we support.

There's a map of our provider dependencies here in
[./providers.jsonc](./providers.jsonc).

The CLI is built with [Deno](https://github.com/denoland/deno) and supposedly
the package ecosystem is such that unused code cannot be tree-shaken, because
static analysis cannot determine whether there are side-effects from imports.
[I'd love to be wrong about this](https://github.com/polyseam/cndi/issues/929).

One side effect of this is that the CLI is quite large, and the user has to
download the entire CLI to use it.

The intended [cdktf](https://developer.hashicorp.com/terraform/cdktf) pattern is
actually to pull down only the packages you need using their `cdktf get`
command.

CNDI includes every
[Terraform Provider](https://registry.terraform.io/browse/providers) we support,
where each cloud's Provider
[eg. aws](https://registry.terraform.io/providers/hashicorp/aws/latest) includes
the entire cloud's API.

Terraform also provides an interface called a
[Terraform Module](https://developer.hashicorp.com/terraform/language/modules)
(not to be confused with a npm package). A module is a collection of resources
that can be used to create a specific piece of infrastructure. For example, the
[aws-eks module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
can be used to create an AWS EKS.

## limitations of the current state of cndi

1. The CLI is large, especially relative to it's _potential_ size (we've seen it
   smaller with comparable functionality by writing out Terraform objects with
   no CDKTF API, see
   [v1.16.0](https://github.com/polyseam/cndi/tree/v1.16.0/src/outputs/terraform/aws-eks))
2. The `cndi ow` command is slow because it must load all the APIs into memory
3. [Terraform Modules](https://developer.hashicorp.com/terraform/cdktf/concepts/modules)
   have been avoided, locking us out of their often simpler and better supported
   APIs
4. CNDI's extensibility is done through
   [Terraform Passthru](https://github.com/polyseam/cndi/blob/main/docs/terraform-passthru.md)
   which effectively takes the `cndi/terraform/cdk.tf.json` we generate for you,
   then deep merging the object found in your config:
   `cndi_config.yaml[infrastructure][terraform]` object. This means that CNDI
   core developers have access to elegant typescript APIs, but CNDI users deal
   with a workaround when the core funcationality is insufficient.

## solving for the limitations

### `TerraformHclModule`

One option to enable the consumption of
[Terraform Modules](https://developer.hashicorp.com/terraform/cdktf/concepts/modules#:~:text=The%20following%20example%20uses%20TerraformHclModule%20to%20import%20an%20AWS%20module.s)
is to use the `TerraformHclModule` class in the CDKTF API. This is really an
escape hatch because it cannot provide Type Safety or Intellisense. This may not
seem like a big thing, but if we wanted to write Terraform without the CDKTF API
and types, we could just write Terraform JSON and manipulate the JSON directly
[like we used to!](https://github.com/polyseam/cndi/tree/v1.16.0/src/outputs/terraform/aws-eks).

Really this solution isn't _that regressive_ because the cdktf modules are still
better than string interpolation, but not by much.

### @cndi/cdktf

The next solution is why we are all here.

We know that there are some CDKTF modules we would like to use, like the
[aws-eks module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest).
We know that to get that module with intellisence we need to call `cdktf get`
with some terraform package metadata present specifying the versions to
download.

What if we maintain a repository and build system which generates the CDKTF
packages we rely on and publishes them to a registry?

This is the goal of `@cndi/cdktf`. It has a JSON configuration file which
specifies the modules we want to build, and the versions we want to build them
at. It will then build the modules and publish them to a registry.

Importing these modules is then just a matter of importing them after they have
been built into the CNDI CLI.

This solution _does not_ change the architecture by pulling the packages from
local disk at runtime.

CNDI still pulls in all the required terraform modules at build time, but it
does so using vendored versions of the modules which are built and published by
`@cndi/cdktf` using the `cdktf` toolkit.

It is not yet clear if this vendoring and publishing process can also do
tree-shaking and code-splitting, but it seems plausible.

This solution doesn't expose a better API as a successor to Terraform
Passthrough. The user still doesn't have access to the rich expressivity of the
CDKTF API. Maybe we tackle this in CNDI v3. Maybe that typescript-centric API is
also synergistic with Typescript FaaS?
