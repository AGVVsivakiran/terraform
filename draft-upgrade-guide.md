# Draft: Upgrading to Terraform v0.13

---

> *WARNING:* This is an early draft of the upgrade guide that we intend to
> publish along with the final v0.13.0 release. It is written for an audience
> that is intending to upgrade to v0.13 and keep using it, but we do not
> recommend doing that for any production systems during the v0.13.0 beta
> period.

---

Terraform v0.13 is a major release and thus includes some changes that
you'll need to consider when upgrading. This guide is intended to help with
that process.

The goal of this guide is to cover the most common upgrade concerns and
issues that would benefit from more explanation and background. The exhaustive
list of changes will always be the
[Terraform Changelog](https://github.com/hashicorp/terraform/blob/master/CHANGELOG.md).
After reviewing this guide, we recommend reviewing the Changelog to check for
specific notes about less-commonly-used features.

This guide focuses on changes from v0.12 to v0.13. Terraform supports upgrade
tools and features only for one major release upgrade at a time, so if you are
currently using a version of Terraform prior to v0.12 please upgrade through
the latest minor releases of all of the intermediate versions first, reviewing
the previous upgrade guides for any considerations that may be relevant to you.

In particular, Terraform v0.13 no longer includes the `terraform 0.12upgrade`
command for automatically migrating module source code from v0.11 to v0.12
syntax. If your modules are written for v0.11 and earlier you may need to
upgrade their syntax using the latest minor release of Terraform v0.12 before
using Terraform v0.13.

---

If you run into any problems during upgrading that are not addressed by the
information in this guide, please feel free to start a topic in
[The Terraform community forum](https://discuss.hashicorp.com/c/terraform-core),
describing the problem you've encountered in enough detail that other readers
may be able to reproduce it and offer advice.

---

Upgrade guide sections:

* [Explicit Provider Source Locations](#explicit-provider-source-locations)
* [New Filesystem Layout for Local Copies of Providers](#new-filesystem-layout-for-local-copies-of-providers)
  * [Special considerations for in-house providers](#in-house-providers)
* [Destroy-time provisioners may not refer to other resources](#destroy-time-provisioners-may-not-refer-to-other-resources)
* [Data resource reads can no longer be disabled by `-refresh=false`](#data-resource-reads-can-no-longer-be-disabled-by--refresh-false)

## Explicit Provider Source Locations

Prior versions of Terraform have supported automatic provider installation only
for providers packaged and distributed by HashiCorp. Providers built by the
community have previously required manual installation by extracting their
distribution packages into specific local filesystem locations.

Terraform v0.13 introduces a new hierarchical namespace for providers that
allows specifying both HashiCorp-maintained and community-maintained providers
as dependencies of a module, with community providers distributed from other
namespaces on [Terraform Registry](https://registry.terraform.io/) from a
third-party provider registry.

In order to establish the hierarchical namespace, Terraform now requires
explicit source information for any providers that are not HashiCorp-maintained,
using a new syntax in the `required_providers` nested block inside the
`terraform` configuration block:

```hcl
terraform {
  required_providers {
    azurerm = {
      # The "hashicorp" namespace is the new home for the HashiCorp-maintained
      # provider plugins.
      #
      # source is not required for the hashicorp/* namespace as a measure of
      # backward compatibility for commonly-used providers, but recommended for
      # explicitness.
      source  = "hashicorp/azurerm"
      version = "~> 2.12"
    }
    newrelic = {
      # source is required for providers in other namespaces, to avoid ambiguity.
      source  = "newrelic/newrelic"
      version = "~> 2.1.1"
    }
  }
}
```

If you are using providers that now require an explicit source location to be
specified, `terraform init` will produce an error like the following:

```
Error: Failed to install providers

Could not find required providers, but found possible alternatives:

  hashicorp/datadog -> terraform-providers/datadog
  hashicorp/fastly -> terraform-providers/fastly

If these suggestions look correct, upgrade your configuration with the
following command:
    terraform 0.13upgrade
```

As mentioned in the error message, Terraform v0.13 includes an automatic
upgrade command
`terraform 0.13upgrade`
that is able to automatically generate source addresses for unlabelled
providers by consulting the same lookup table that was previously used for
Terraform v0.12 provider installation. This command will automatically modify
the configuration of your current module, so you can use the features of your
version control system to inspect the proposed changes before committing them.

We recommend running `terraform 0.13upgrade` even if you don't see the message,
because it will generate the recommended explicit source addresses for
providers in the "hashicorp" namespace.

For some introductory information on provider dependencies, see
[the beta guide for provider sources](https://github.com/hashicorp/terraform/tree/guide-v0.13-beta/provider-sources).
Fuller instructions will follow on the main Terraform documentation website
after the v0.13.0 release, including some guidance on how you can write
modules that are compatible with both v0.12 and v0.13 during a transitional
period.

Each module must declare its own set of provider requirements, so if you have
a configuration which calls other modules then you'll need to run this upgrade
command for each module separately.

After you've added explicit provider source addresses to your configuration,
run `terraform init` again to re-run the provider installer.

---

**Action:** Either run `terraform 0.13upgrade` for each of your modules, or manually update the provider declarations to use explicit source addresses.

---

## New Filesystem Layout for Local Copies of Providers

As part of introducing the hierarchical provider namespace discussed in the
previous section, Terraform v0.13 also introduces a new hierarchical directory
structure for manually-installed providers in the local filesystem.

If you use local copies of official providers or if you use custom in-house
providers that you have installed manually, you will need to adjust your local
directories to use the new directory structure.

The previous layout was a single directory per target platform containing
various executable files named with the prefix `terraform-provider`, like
`linux_amd64/terraform-provider-google_v2.0.0`. The new expected location for the
Google Cloud Platform provider for that target platform within one of the local
search directories would be the following:

```
registry.terraform.io/hashicorp/google/2.0.0/linux_amd64/terraform-provider-google_v2.0.0
```

The `registry.terraform.io` above is the hostname of the registry considered
to be the origin for this provider. The provider source address
`hashicorp/google` is a shorthand for `registry.terraform.io/hashicorp/google`,
and the full, explicit form is required for a local directory.

Note that the version number given as a directory name must be written _without_
the "v" prefix that tends to be included when a version number is used as part
of a git branch name. If you include that prefix, Terraform will not recognize
the directory as containing provider packages.

As before, the recommended default location for locally-installed providers
is one of the following, depending on which operating system you are running
Terraform under:

* Windows: `%APPDATA%\terraform.d\plugins`
* All other systems: `~/.terraform.d/plugins`

Terraform v0.13 introduces some additional options for customizing where
Terraform looks for providers in the local filesystem. For more information on
those new options, see [Provider Installation](/docs/commands/cli-config.html#provider-installation).

If you use only providers that are automatically installable from Terraform
provider registries but still want to avoid Terraform re-downloading them from
registries each time, Terraform v0.13 includes
the `terraform providers mirror` command
which you can use to automatically populate a local directory based on the
requirements of the current configuration file:

```
terraform providers mirror ~/.terraform.d/plugins
```

---

**Action:** If you use local copies of official providers rather than installing them automatically from Terraform Registry, adopt the new expected directory structure for your local directory either by running `terraform providers mirror` or by manually reorganizing the existing files.

---

### In-house Providers

If you use an in-house provider that is not available from an upstream registry
at all, after upgrading you will see an error similar to the following:

```
- Finding latest version of hashicorp/happycloud...

Error: Failed to install provider

Error while installing hashicorp/happycloud: provider registry
registry.terraform.io does not have a provider named
registry.terraform.io/hashicorp/happycloud
```

Terraform assumes that a provider without an explicit source address belongs
to the "hashicorp" namespace on `registry.terraform.io`, which is not true
for your in-house provider. Instead, you can use any domain name under your
control to establish a _virtual_ source registry to serve as a separate
namespace for your local use. For example:

```
terraform.example.com/awesomecorp/happycloud/v1.0.0/linux_amd64/terraform-provider-happycloud_v1.0.0
```

You can then specify explicitly the requirement for that in-house provider
in your modules, using the requirement syntax discussed in the previous section:

```hcl
terraform {
  required_providers {
    happycloud = {
      source  = "terraform.example.com/awesomecorp/happycloud"
      version = "1.0.0"
    }
  }
}
```

If you wish, you can later run your own Terraform provider registry at the
specified hostname as an alternative to local installation, without any further
modifications to the above configuration. However, we recommend tackling that
only after your initial upgrade using the new local filesystem layout.

---

**Action:** If you use in-house providers that are not installable from a provider registry, assign them a new source address under a domain name you control and update your modules to specify that new source address.

---

If your configuration using one or more in-house providers has existing state
snapshots that include resources belonging to those providers, you'll also need
to perform a one-time migration of the provider references in the state, so
Terraform can understand them as belonging to your in-house providers rather
than to providers in the public Terraform Registry. If you are in this
situation, `terraform init` will produce the following error message after
you complete the configuration changes described above:

```
Error: Failed to query available provider packages

Could not retrieve the list of available versions for provider -/happycloud:
provider registry registry.terraform.io does not have a provider named
registry.terraform.io/-/happycloud
```

Provider source addresses starting with `registry.terraform.io/-/` are a special
way Terraform marks legacy addresses where the true namespace is unknown.
For providers that were automatically-installable in Terraform 0.12, Terraform
0.13 can automatically determine the new addresses for these using a lookup
table in the public Terraform Registry, but for in-house providers you will
need to provide the appropriate mapping manually.

The `terraform state replace-provider` subcommand allows re-assigning provider
source addresses recorded in the Terraform state, and so we can use this
command to tell Terraform how to reinterpret the "legacy" provider addresses
as properly-namespaced providers that match with the provider source addresses
in the configuration.

---

**Warning:** The `terraform state replace-provider` subcommand, like all of the `terraform state` subcommands, will create a new state snapshot and write it to the configured backend. After the command succeeds the latest state snapshot will use syntax that Terraform v0.12 cannot understand, so you should perform this step only when you are ready to permanently upgrade to Terraform v0.13.

---

```
terraform state replace-provider 'registry.terraform.io/-/happycloud' 'terraform.example.com/awesomecorp/happycloud'
```

The command above asks Terraform to update any resource instance in the state
that belongs to a legacy (non-namespaced) provider called "happycloud" to
instead belong to the fully-qualified source address
`terraform.example.com/awesomecorp/happycloud`.

Whereas the configuration changes for provider requirements are made on a
per-module basis, the Terraform state captures data from throughout the
configuration (all of the existing module instances) and so you only need to
run `terraform state replace-provider` once per configuration.

Running `terraform init` again after completing this step should cause
Terraform to attempt to install `terraform.example.com/awesomecorp/happycloud`
and to find it in the local filesystem directory you populated in an earlier
step.

---

**Action:** If you use in-house providers that are not installable from a provider registry and your existing state contains resource instances that were created with any of those providers, use the `terraform state replace-provider` command to update the state to use the new source addressing scheme only once you are ready to commit to your v0.13 upgrade. (Terraform v0.12 cannot parse a state snapshot that was created by this command.)

---

## Destroy-time provisioners may not refer to other resources

Destroy-time provisioners allow introducing arbitrary additional actions into
the destroy phase of the resource lifecycle, but in practice the design of this
feature was flawed because it created the possibility for a destroy action
of one resource to depend on a create or update action of another resource,
which often leads either to dependency cycles or to incorrect behavior due to
unsuitable operation ordering.

In order to retain as many destroy-time provisioner capabilities as possible
while addressing those design flaws, Terraform v0.12.18 began reporting
deprecation warnings for any `provisioner` block setting `when = destroy` whose
configuration refers to any objects other than `self`, `count`, and `each`.

Addressing the flaws in the destroy-time provisioner design was a pre-requisite
for new features in v0.13 such as module `depends_on`, so Terraform v0.13
concludes the deprecation cycle by making such references now be fatal errors:

```
Error: Invalid reference from destroy provisioner

Destroy-time provisioners and their connection configurations may only
reference attributes of the related resource, via 'self', 'count.index',
or 'each.key'.

References to other resources during the destroy phase can cause dependency
cycles and interact poorly with create_before_destroy.
```

Some existing modules using resource or other references inside destroy-time
provisioners can be updated by placing the destroy-time provisioner inside a
`null_resource` resource and copying any data needed at destroy time into
the `triggers` map to be accessed via `self`:

```hcl
resource "null_resource" "example" {
  triggers = {
    instance_ip_addr = aws_instance.example.private_ip
  }

  provisioner "remote-exec" {
    when = destroy

    connection {
      host = self.triggers.instance_ip_addr
      # ...
    }

    # ...
  }
}
```

In the above example, the `null_resource.example.triggers` map is effectively
acting as a temporary "cache" for the instance's private IP address to
guarantee that a value will be available when the provisioner runs, even if
the `aws_instance.example` object itself isn't currently available.
The provisioner's `connection` configuration can refer to that value via
`self`, whereas referring directly to `aws_instance.example.private_ip` in that
context is forbidden.

[Provisioners are a last resort](/docs/provisioners/#provisioners-are-a-last-resort),
so we recommend avoiding both create-time and destroy-time provisioners wherever
possible. Other options for destroy-time actions include using `systemd` to
run commands within your virtual machines during shutdown or using virtual
machine lifecycle hooks provided by your chosen cloud computing platform,
both of which can help ensure that the shutdown actions are taken even if the
virtual machine is terminated in an unusual way.

---

**Action:** If you encounter the "Invalid reference from destroy provisioner" error message after upgrading, reorganize your destroy-time provisioners to depend only on self-references, and consider other approaches if possible to avoid using destroy-time provisioners at all.

---

## Data resource reads can no longer be disabled by `-refresh=false`

In Terraform v0.12 and earlier, Terraform would read the data for data
resources during the "refresh" phase of `terraform plan`, which is the same
phase where Terraform synchronizes its state with any changes made to
remote objects.

An important prerequisite for properly supporting `depends_on` for both
data resources and modules containing data resources was to change the data
resource lifecycle to now read data during the _plan_ phase, so that
dependencies on managed resources could be properly respected.

If you were previously using `terraform plan -refresh=false` or
`terraform apply -refresh=false` to disable the refresh phase, you will find
that under Terraform 0.13 this will continue to disable synchronization of
managed resources (declared with `resource` blocks) but will no longer
disable the reading of data resources (declared with `data` blocks).

---

Updating the data associated with data resources is crucial to producing an
accurate plan, and so there is no replacement mechanism in Terraform v0.13
to restore the previous behavior.

---