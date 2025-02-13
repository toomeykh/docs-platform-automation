##  Using Your Credhub
Credhub can be used to store secure properties that you don't want committed into a config file.
Within your pipeline, the config file can then reference that Credhub value for runtime evaluation.

An example workflow would be storing an SSH key.

1. Authenticate with your credhub instance.
2. Generate an ssh key: `credhub generate --name="/private-foundation/opsman_ssh_key" --type=ssh`
3. Create an [Ops Manager configuration][opsman-config] file that references the name of the property.

```yaml
opsman-configuration:
  azure:
    ssh_public_key: ((opsman_ssh_key.public_key))
```

4. Configure your pipeline to use the [credhub-interpolate] task.
   It takes an input `files`, which should contain your configuration file from (3).

   The declaration within a pipeline might look like:

```yaml
jobs:
- name: example-job
  plan:
  - get: platform-automation-tasks
  - get: platform-automation-image
  - get: config
  - task: credhub-interpolate
    image: platform-automation-image
    file: platform-automation-tasks/tasks/credhub-interpolate.yml
    input_mapping:
      files: config
    params:
      # depending on credhub configuration
      # ether CA cert or secret are required
      CREDHUB_CA_CERT: ((credhub_ca_cert))
      CREDHUB_SECRET: ((credhub_secret))

      # all required
      CREDHUB_CLIENT: ((credhub_client))
      CREDHUB_SERVER: ((credhub_server))
      PREFIX: /private-foundation
      SKIP_MISSING: true  
```

Notice the `PREFIX` has been set to `/private-foundation`, the path prefix defined for your cred in (2).
This allows the config file to have values scoped, for example, per foundation.
`params` should be filled in by the credhub created with your Concourse instance.

!!! info
    You can set the param `SKIP_MISSING:false` to enforce strict checking of 
    your vars files during intrpolation. This is true by default to support 
    credential management from multiple sources. For more information, see the 
    [Multiple Sources](#multiple-sources) section.

This task will reach out to the deployed credhub and fill in your entry references and return an output
named `interpolated-files` that can then be read as an input to any following tasks.

Our configuration will now look like

```yaml
opsman-configuration:
 azure:
   ssh_public_key: ssh-rsa AAAAB3Nz...
```

!!! info 
    If using this you need to ensure the concourse worker can talk to credhub so depending
    on how you deployed credhub and/or worker this may or may not be possible.
    This inverts control that now workers need to access credhub vs
    default is atc injects secrets and passes them to the worker.

## Defining Multiline Certificates and Keys in Config Files
There are three ways to include certificates in the yaml files that are used by Platform Automation tasks.

1. Direct inclusion in yaml file

    ```yaml
    # An incomplete base.yml response from om staged-config
    product-name: cf

    product-properties:
      .uaa.service_provider_key_credentials:
        value:
          cert_pem: |
            -----BEGIN CERTIFICATE-----
            ...<Some Cert>...
            -----END CERTIFICATE-----
          private_key_pem: |
            -----BEGIN RSA PRIVATE KEY-----
            ...<Some Private Key>...
            -----END RSA PRIVATE KEY-----

      .properties.networking_poe_ssl_certs:
        value:
          -
            certificate:
              cert_pem: |
                -----BEGIN CERTIFICATE-----
                ...<Some Cert>...
                -----END CERTIFICATE-----
              private_key_pem: |
                -----BEGIN RSA PRIVATE KEY-----
                ...<Some Private Key>...
                -----END RSA PRIVATE KEY-----
    ```

1. Secrets Manager reference in yaml file

    ```yaml
    # An incomplete base.yml
    product-name: cf

    product-properties:
      .uaa.service_provider_key_credentials:
        value:
          cert_pem: ((uaa_service_provider_key_credentials.certificate))
          private_key_pem: ((uaa_service_provider_key_credentials.private_key))

      .properties.networking_poe_ssl_certs:
        value:
          -
            certificate:
              cert_pem: ((networking_poe_ssl_certs.certificate))
              private_key_pem: ((networking_poe_ssl_certs.private_key))
    ```

    This example assumes the use of Credhub.

    Credhub supports a `--type=certificate` credential type
    which allows you to store a certificate and private key pair under a single name.
    The cert and key can be stored temporarily in local files
    or can be passed directly on the command line.

    An example of the file storage method:

    ```bash
    credhub set --type=certificate \
      --name=uaa_service_provider_key_credentials \
      --certificate=./cert.pem \
      --private=./private.key
    ```

1. Using vars files

    Vars files are a mix of the two previous methods.
    The cert/key is defined inline in the vars file:

    ```yaml
    #vars.yml
    uaa_service_provider_key_credentials_cert_pem: |
      -----BEGIN CERTIFICATE-----
      ...<Some Cert>...
      -----END CERTIFICATE-----
    uaa_service_provider_key_credentials_private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...<Some Private Key>...
      -----END RSA PRIVATE KEY-----
    networking_poe_ssl_certs_cert_pem: |
      -----BEGIN CERTIFICATE-----
      ...<Some Cert>...
      -----END CERTIFICATE-----
    networking_poe_ssl_certs_private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...<Some Private Key>...
      -----END RSA PRIVATE KEY-----
    ```

    and referenced as a `((parameter))` in the `base.yml`

    ```yaml
    # An incomplete base.yml
    product-name: cf

    product-properties:
      .uaa.service_provider_key_credentials:
        value:
          cert_pem: ((uaa_service_provider_key_credentials_cert_pem))
          private_key_pem: ((uaa_service_provider_key_credentials_private_key))

      .properties.networking_poe_ssl_certs:
        value:
          -
            certificate:
              cert_pem: ((networking_poe_ssl_certs_cert_pem))
              private_key_pem: ((networking_poe_ssl_certs_private_key))
    ```

## Storing values for Multi-foundation
### Credhub
In the example above, `bosh int` did not replace the ((placeholder_credential)): `((cloud_controller_encrypt_key.secret))`.
For security, values such as secrets and keys should not be saved off in static files (such as an ops file). In order to
rectify this, you can use a secret management tool, such as Credhub, to sub in the necessary values to the deployment
manifest.  

Let's assume basic knowledge and understanding of the
[`credhub-interpolate`][credhub-interpolate] task described in the [Secrets Handling][secrets-handling] section
of the documentation.

For multiple foundations, [`credhub-interpolate`][credhub-interpolate] will work the same, but `PREFIX` param will
differ per foundation. This will allow you to keep your `base.yml` the same for each foundation with the same
((placeholder_credential)) reference. Each foundation will require a separate [`credhub-interpolate`][credhub-interpolate]
task call with a unique prefix to fill out the missing pieces of the template.

### Vars Files
Vars files can be used for your [Secrets Handling][secrets-handling].

Take the example below:

{% include ".cf-partial-config.md" %}

In our first foundation, we have the following `vars.yml`, optional for the [`configure-product`][configure-product] task.
```yaml
# vars.yml
cloud_controller_encrypt_key.secret: super-secret-encryption-key
cloud_controller_apps_domain: cfapps.domain.com
```

The `vars.yml` could then be passed to [`configure-product`][configure-product] with `base.yml` as the config file.
The task will then sub the `((cloud_controller_encrypt_key.secret))` and `((cloud_controller_apps_domain))` 
specified in `vars.yml` and configure the product as normal.

An example of how this might look in a pipeline(resources not listed):
```yaml
jobs:
- name: configure-product
  plan:
  - aggregate:
    - get: platform-automation-image
      params:
        unpack: true
    - get: platform-automation-tasks
      params:
        unpack: true
    - get: configuration
    - get: variable
  - task: configure-product
    image: platform-automation-image
    file: platform-automation-tasks/tasks/configure-product.yml
    input_mapping:
      config: configuration
      env: configuration
      vars: variable
    params:
      CONFIG_FILE: base.yml
      VARS_FILES: vars.yml
      ENV_FILE: env.yml
```

### Multiple Sources

Both vars files and credhub may be used to interpolate variables into `base.yml`.
Using the same example from above: 

{% include ".cf-partial-config.md" %}

We have one parametrized variable that is secret and might not want to have stored in 
a plain text vars file, `((cloud_controller_encrypt_key.secret))`, but `((cloud_controller_apps_domain))` 
is fine in a vars file. In order to support a `base.yml` with credentials from multiple sources (i.e. 
credhub and vars files), you will need to `SKIP_MISSING: true` in the [`credhub-interpolate`][credhub-interpolate] task.
This is enabled by default by the `credhub-interpolate` task.

The workflow would be the same as [Credhub](#credhub), but when passing the interpolated `base.yml` as a config into the
next task, you would add in a [Vars File](#vars-files) to fill in the missing variables.

An example of how this might look in a pipeline (resources not listed), assuming:

- The `((base.yml))` above 
- `((cloud_controller_encrypt_key.secret))` is stored in credhub
- `((cloud_controller_apps_domain))` is stored in `director-vars.yml` 

```yaml
jobs:
- name: example-credhub-interpolate
  plan:
  - get: platform-automation-tasks
  - get: platform-automation-image
  - get: config
  - task: credhub-interpolate
    image: platform-automation-image
    file: platform-automation-tasks/tasks/credhub-interpolate.yml
    input_mapping:
      files: config
    params:
      # depending on credhub configuration
      # ether Credhub CA cert or Credhub secret are required
      CREDHUB_CA_CERT: ((credhub_ca_cert))
      CREDHUB_SECRET: ((credhub_secret))

      # all required
      CREDHUB_CLIENT: ((credhub_client))
      CREDHUB_SERVER: ((credhub_server))
      PREFIX: /private-foundation
      SKIP_MISSING: true
- name: example-configure-director
  plan:
  - get:   
  - task: configure-director
    image: platform-automation-image
    file: platform-automation-tasks/tasks/configure-director.yml
    params:
      VARS_FILES: vars/director-vars.yml
      ENV_FILE: env/env.yml
      DIRECTOR_CONFIG_FILE: config/director.yml
```

## Multiple Key Lookups in a Single Task

When using Credhub in a single foundation or multi-foundation manner, 
we want to avoid duplicating identical credentials
(duplication makes credential rotation harder). 

In order to have Credhub read in credentials from multiple paths
(not relative to your `PREFIX`), 
you must provide the absolute path to any credentials 
not in your relative path.

For example, using an alternative `base.yml`:
```yaml
# An incomplete yaml response from om staged-config
product-name: cf

product-properties:
  .cloud_controller.apps_domain:
    value: ((cloud_controller_apps_domain))
  .cloud_controller.encrypt_key:
    value:
      secret: ((/alternate_prefix/cloud_controller_encrypt_key.secret))
  .properties.security_acknowledgement:
    value: X
  .properties.cloud_controller_default_stack:
    value: default
```

Let's say in our `job`, we define the prefix as "foundation1".
The parameterized values in the example above will be interpolated as follows:

`((cloud_controller_apps_domain))` uses a relative path for Credhub.
When running `credhub-interpolate`, the task will prepend the `PREFIX`.
This value is stored in Credhub as `/foundation1/cloud_controller_apps_domain`. 

`((/alternate_prefix/cloud_controller_encrypt_key.secret))` (note the leading slash)
uses an absolute path for Credhub. 
When running `credhub-interpolate`, the task will not prepend the prefix.
This value is stored in Credhub at it's absolute path `/alternate_prefix/cloud_controller_encrypt_key.secret`.

Any value with a leading `/` slash will never use the `PREFIX`
to look up values in Credhub. 
Therefore, you can have multiple key lookups in a single interpolate task. 


{% with path="../" %}
    {% include ".internal_link_url.md" %}
{% endwith %}
{% include ".external_link_url.md" %}