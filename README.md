# Base Image Factory

This provisions a Base Image from a previously created Root Image.

#### A note on terminology

We use `Root Image` to mean a completely unprovisioned bare image with nothing beyond a basic OS install. We use `Base Image` to mean a partially provisioned image with services required by all operational servers, e.g. monitoring, log aggregation, telemetry, etc.

## Setup

### Basic Tooling

First setup Homebrew, and make sure everything is up to date.

Clone this repo and `cd` into it.  Run the following to set up tools:

```bash
brew bundle
```

When setting things up, you will need to download a copy of the fork of Packer we use from here: <https://github.com/AlexSc/packer/releases/tag/v1.7.5-dev3>

Unzip and put the file in `~/bin`.

### Secrets

Copy `secrets.auto.pkrvars.hcl.template` to `secrets.auto.pkrvars.hcl`, and populate all the variables listed within.

## Image Specifications

When this image provides the option to include additional configuration files in a directory, file names must be prefixed with two digits and end in .conf. The prefixes 00 to 29 and 90 to 95 are reserved for use by this image.

### TEAK_SERVICE

The Base Image configures systemd to provide a TEAK_SERVICE environment variable to all systemd services with names starting with `teak-`. By default TEAK_SERVICE will be set to the name of the base image AMI. In non-AMI environments, TEAK_SERVICE will be set to "unknown". To modify this create a configuration file in /etc/systemd/system/teak-.service.d/ with the contents

```
[Service]
Environment="TEAK_SERVICE={{service_name}}"
```

### teak-init.target

The Base Image provides teak-init.target, which will not be active until all services provided by the Base Image are available. Downstream services should set `After=teak-init.target` in their unit configurations.

### Fluentd

The Base Image provides [Fluentd](https://www.fluentd.org) as teak-log-collector, with the following defaults:

- systemd, cloudinit, fluentd, and configurator logs are tailed under ancillary.{process}
- ancillary logs are outputted to cloudwatch_logs under /fb/server/{{ server_environment }}/ancillary/{{ process_name }}:{{ service_name }}.{{ hostname }}
- logs with the service.default tag will be outputted to /fb/server/{{ server_environment }}/service/{{ service_name }}:{{ service_name }}.{{ hostname }}
- Downstream images may add additional configuration for fluentd in /etc/fluent/conf.d/\*.conf.

Fluentd is enabled by default in this image.

#### Disabling Fluentd

To disable Fluentd at boot, use the following user-data

```yml
#cloud-config
bootcmd:
  - [systemctl, stop, --no-block, teak-log-collector]
```

Be sure to wipe `/var/lib/cloud` after provisioning so that this user-data does not persist to live servers.

It is recommended that Fluentd remain enabled so that the server logs from the build process running be logged to CloudWatch.

### Config O-Mat

The Base Image provides the [config_o_mat](https://github.com/GoCarrot/config_o_mat) as teak-configurator.

teak-configurator is enabled by default in this image.

As the Base Image provides no "metaconfiguration" for the configurator it will not actually do anything.

## Adding a New Language

```bash
cd language_images
cp -Rfp <some_existing_language> <new_language> # Note:  Don't put trailing slashes on the directory names!
ls -la <new_language>/image.pkr.hcl
# You should see something like:
# lrwxr-xr-x  1 jonathonfrisby  staff  24 Feb  8 11:55 node12/image.pkr.hcl -> ../../base_image.pkr.hcl

# If, and only if, the file is _not_ a symlink, then do the following:
cd <new_language>
rm image.pkr.hcl
ln -sfn ../../base_image.pkr.hcl image.pkr.hcl # We want this to be a symlink to the base one!

# Once the directory is set up, with the Packer definition being a symlink:
#
# Edit image.auto.pkrvars.hcl to change `ami_prefix` and `cost_center`.
#
# Edit playbooks as appropriate.
```

### New Ruby Versions

Start from the most recent, relevant Ruby image, copying to a new folder with an appropriate name as per the general instructions.

In the `language_images/rubyXX/playbooks/ruby.yml` file, look for lines that look like this:

```
        RUBY_SERIES: "3.0"
        RUBY_VERSION: "3.0.3"
        RUBY_CHECKSUM: 3586861cb2df56970287f0fd83f274bd92058872d830d15570b36def7f1a92ac
```

Revise these with appropriate values, from the [official website](https://www.ruby-lang.org/en/downloads/).


## Provisioning

To build Debian 11 base AMIs:

```bash
~/bin/packer_1.7.5-dev3_darwin_arm64 init .

aws-vault exec fb -- ~/bin/packer_1.7.5-dev3_darwin_arm64 build --var-file=base_image.auto.pkrvars.hcl --var-file=secrets.auto.pkrvars.hcl -var region=us-east-1 -var build_account_canonical_slug=stage-ci-cd -var use_generated_security_group=true -var cost_center=root_image -timestamp-ui '-except=vagrant.*' base_image.pkr.hcl
```

To build Debian 11 language-specific AMIs, first build a base AMI and then:

```bash
cd language_images/<language>/

~/bin/packer_1.7.5-dev3_darwin_arm64 init .

aws-vault exec fb -- ~/bin/packer_1.7.5-dev3_darwin_arm64 build -var-file=image.auto.pkrvars.hcl -var region=us-east-1 -var build_account_canonical_slug=stage-ci-cd -var use_generated_security_group=true -var cost_center=root_image -timestamp-ui '-except=vagrant.*' image.pkr.hcl
```
