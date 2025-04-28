<!-- BEGIN_TF_DOCS -->
# terraform-azure-avm-utl-ephemeral-credential

## Introduction: Why "No-SSH"?

In modern cloud environments, especially when adopting DevOps best practices and immutable infrastructure models, it is increasingly recommended to avoid direct login access (SSH/RDP) to servers whenever possible. This approach, known as the **"No-SSH" style**, promotes:

- Stronger security by eliminating backdoor access.
- Clear separation between operational management and system state.
- Greater automation and resilience.

Inspired by the following ideas, this module helps implement "No-SSH" or minimal-access cloud infrastructures, especially when you still need to configure credentials during provisioning.

* [Immutable Infrastructure: No SSH](https://cloudcaptain.sh/blog/no-ssh)
* [To ssh, or not to ssh](https://steve-mushero.medium.com/to-ssh-or-not-to-ssh-c294b49298cd)
* [AWS re:Invent 2016: Life Without SSH: Immutable Infrastructure in Production (SAC318)](https://www.youtube.com/watch?v=fEuN5LkXfZk&ab_channel=AmazonWebServices)

## Why This Module Was Created

Traditional Terraform workflows either:

- Statically store credentials in state files (risky!), or
- Rely heavily on persistent SSH key management (contradicting No-SSH philosophy).

This module was created to solve:

- **Secure, ephemeral credential generation** at VM creation time.
- **No persistent secrets** unless explicitly required.
- **Support for rotation awareness** without breaking Terraform's refresh model.
- **Integration with Azure Key Vault** when retrieval is needed securely.

## Supported Usage Scenarios

| Scenario                                    | How to Use                                                                                                                                                       |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Windows VM, No Retrieval                 | Set `var.password`, leave `var.key_vault_password_secret = null`. Password is assigned and then thrown away. No one, including the creator, can later access it. |
| 2. Linux VM, No Retrieval (Public Key)      | Set `var.private_key` to generate a temporary key pair. Use the public key to configure the VM. Private key is discarded.                                        |
| 3. Password Stored in Azure Key Vault       | Set `var.password` and configure `var.key_vault_password_secret`. Password is assigned to VM and securely stored as a Key Vault Secret for later retrieval.      |
| 4. Private Key Storage (Not Supported Here) | This module is **not intended** for private key storage. Please use `azurerm_key_vault_key` or another mechanism if needed.                                      |

## Credential Rotation Support

This module optionally supports a **rotation notification** system via `var.time_rotating`:

- Even without `var.time_rotating`, ephemeral credentials are **regenerated** each time Terraform refreshes them.
- Configuring `var.time_rotating` adds a monotonically increasing **version number** (`value_wo_version`) and a **next rotation timestamp** (`rotation_rfc3339`).
- This is helpful for downstream systems like Azure Key Vault or custom automation to know when to rotate or expire secrets.

> **Note:** `var.time_rotating` **does not control** when the ephemeral password/private key is regenerated — it only creates a rotation metadata mechanism.

## Key Outputs

| Output             | Description                                                    |
| ------------------ | -------------------------------------------------------------- |
| `value_wo`         | The generated password (sensitive).                            |
| `value_wo_version` | Updated when a rotation event happens (increase-only version). |
| `rotation_rfc3339` | RFC3339 timestamp of the next rotation.                        |

## Security Best Practices

- If you must retrieve credentials (e.g., via Key Vault), ensure tight access control.
- Remember that **ephemeral credentials refresh** every Terraform run, aligning with zero-trust principles.

> **Warning:** If `var.key_vault_password_secret` is not set, all generated credentials (passwords and private keys) will be discarded after VM creation and will not be retrievable later.

---

*Designed for security-first infrastructures in the Terraform era.*

<!-- markdownlint-disable MD033 -->
## Requirements

The following requirements are needed by this module:

- <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) (>= 1.10, < 2.0)

- <a name="requirement_azurerm"></a> [azurerm](#requirement\_azurerm) (~> 4.0)

- <a name="requirement_modtm"></a> [modtm](#requirement\_modtm) (~> 0.3)

- <a name="requirement_random"></a> [random](#requirement\_random) (>= 3.7.1, < 4.0)

- <a name="requirement_time"></a> [time](#requirement\_time) (>= 0.7.1, < 1.0)

## Resources

The following resources are used by this module:

- [azurerm_key_vault_secret.password](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_secret) (resource)
- [modtm_telemetry.telemetry](https://registry.terraform.io/providers/azure/modtm/latest/docs/resources/telemetry) (resource)
- [random_uuid.telemetry](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/uuid) (resource)
- [time_rotating.this](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/rotating) (resource)
- [azurerm_client_config.telemetry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/client_config) (data source)
- [modtm_module_source.telemetry](https://registry.terraform.io/providers/azure/modtm/latest/docs/data-sources/module_source) (data source)

<!-- markdownlint-disable MD013 -->
## Required Inputs

No required inputs.

## Optional Inputs

The following input variables are optional (have default values):

### <a name="input_enable_telemetry"></a> [enable\_telemetry](#input\_enable\_telemetry)

Description: This variable controls whether or not telemetry is enabled for the module.  
For more information see <https://aka.ms/avm/telemetryinfo>.  
If it is set to false, then no telemetry will be collected.

Type: `bool`

Default: `true`

### <a name="input_key_vault_password_secret"></a> [key\_vault\_password\_secret](#input\_key\_vault\_password\_secret)

Description: - `content_type` - (Optional) Specifies the content type for the Key Vault Secret.
- `expiration_date` - (Optional) Expiration UTC datetime (Y-m-d'T'H:M:S'Z').
- `key_vault_id` - (Required) The ID of the Key Vault where the Secret should be created. Changing this forces a new resource to be created.
- `not_before_date` - (Optional) Key not usable before the provided UTC datetime (Y-m-d'T'H:M:S'Z').
- `tags` - (Optional) A mapping of tags to assign to the resource.
- `value` - (Optional) Specifies the value of the Key Vault Secret. Changing this will create a new version of the Key Vault Secret.

---
`timeouts` block supports the following:
- `create` - (Defaults to 30 minutes) Used when creating the Key Vault Secret.
- `delete` - (Defaults to 30 minutes) Used when deleting the Key Vault Secret.
- `read` - (Defaults to 30 minutes) Used when retrieving the Key Vault Secret.
- `update` - (Defaults to 30 minutes) Used when updating the Key Vault Secret.

Type:

```hcl
object({
    content_type    = optional(string)
    expiration_date = optional(string)
    key_vault_id    = string
    name            = string
    not_before_date = optional(string)
    tags            = optional(map(string))
    timeouts = optional(object({
      create = optional(string)
      delete = optional(string)
      read   = optional(string)
      update = optional(string)
    }))
  })
```

Default: `null`

### <a name="input_password"></a> [password](#input\_password)

Description: - `keepers` -
- `length` -
- `lower` - (Boolean) Include lowercase alphabet characters in the result. Default value is `true`.
- `min_lower` - (Number) Minimum number of lowercase alphabet characters in the result. Default value is `0`.
- `min_numeric` - (Number) Minimum number of numeric characters in the result. Default value is `0`.
- `min_special` - (Number) Minimum number of special characters in the result. Default value is `0`.
- `min_upper` - (Number) Minimum number of uppercase alphabet characters in the result. Default value is `0`.
- `numeric` - (Boolean) Include numeric characters in the result. Default value is true. If `numeric`, `upper`, `lower`, and `special` are all configured, at least one of them must be set to `true`.
- `override_special` - (String) Supply your own list of special characters to use for string generation. This overrides the default character list in the special argument. The `special` argument must still be set to true for any overwritten characters to be used in generation.
- `special` - (Boolean) Include special characters in the result. These are `!@#$%&*()-_=+[]{}<>:?`. Default value is `true`.
- `upper` - (Boolean) Include uppercase alphabet characters in the result. Default value is `true`.

Type:

```hcl
object({
    keepers          = optional(map(string))
    length           = number
    lower            = optional(bool)
    min_lower        = optional(number)
    min_numeric      = optional(number)
    min_special      = optional(number)
    min_upper        = optional(number)
    number           = optional(bool)
    numeric          = optional(bool)
    override_special = optional(string)
    special          = optional(bool)
    upper            = optional(bool)
  })
```

Default: `null`

### <a name="input_private_key"></a> [private\_key](#input\_private\_key)

Description: - `algorithm` - (String) Name of the algorithm to use when generating the private key. Currently-supported values are: `RSA`, `ECDSA`, `ED25519`.
- `ecdsa_curve` - (String) When `algorithm` is `ECDSA`, the name of the elliptic curve to use. Currently-supported values are: `P224`, `P256`, `P384`, `P521`. (default: `P224`).
- `rsa_bits` - (Number) When `algorithm` is RSA, the size of the generated RSA key, in bits (default: `2048`).

Type:

```hcl
object({
    algorithm   = string
    ecdsa_curve = optional(string)
    rsa_bits    = optional(number)
  })
```

Default: `null`

### <a name="input_time_rotating"></a> [time\_rotating](#input\_time\_rotating)

Description: - `rfc3339` - (String) Base timestamp in [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339#section-5.8) format (see [RFC3339 time string](https://tools.ietf.org/html/rfc3339#section-5.8) e.g., `YYYY-MM-DDTHH:MM:SSZ`). Defaults to the current time.
- `rotation_days` - (Number) Number of days to add to the base timestamp to configure the rotation timestamp. When the current time has passed the rotation timestamp, the resource will trigger recreation. At least one of the 'rotation\_' arguments must be configured.
- `rotation_hours` - (Number) Number of hours to add to the base timestamp to configure the rotation timestamp. When the current time has passed the rotation timestamp, the resource will trigger recreation. At least one of the 'rotation\_' arguments must be configured.
- `rotation_minutes` - (Number) Number of minutes to add to the base timestamp to configure the rotation timestamp. When the current time has passed the rotation timestamp, the resource will trigger recreation. At least one of the 'rotation\_' arguments must be configured.
- `rotation_months` - (Number) Number of months to add to the base timestamp to configure the rotation timestamp. When the current time has passed the rotation timestamp, the resource will trigger recreation. At least one of the 'rotation\_' arguments must be configured.
- `rotation_rfc3339` - (String) Configure the rotation timestamp with an [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339#section-5.8) format of the offset timestamp. When the current time has passed the rotation timestamp, the resource will trigger recreation. At least one of the 'rotation\_' arguments must be configured.
- `rotation_years` - (Number) Number of years to add to the base timestamp to configure the rotation timestamp. When the current time has passed the rotation timestamp, the resource will trigger recreation. At least one of the 'rotation\_' arguments must be configured.
- `triggers` - (Map of String) Arbitrary map of values that, when changed, will trigger a new base timestamp value to be saved. These conditions recreate the resource in addition to other rotation arguments. See [the main provider documentation](../index.md) for more information.

Type:

```hcl
object({
    rfc3339          = optional(string)
    rotation_days    = optional(number)
    rotation_hours   = optional(number)
    rotation_minutes = optional(number)
    rotation_months  = optional(number)
    rotation_rfc3339 = optional(string)
    rotation_years   = optional(number)
    triggers         = optional(map(string))
  })
```

Default: `null`

## Outputs

The following outputs are exported:

### <a name="output_password_bcrypt_hash"></a> [password\_bcrypt\_hash](#output\_password\_bcrypt\_hash)

Description: (String, Sensitive) A bcrypt hash of the generated random string. NOTE: If the generated random string is greater than 72 bytes in length, `bcrypt_hash` will contain a hash of the first 72 bytes.

### <a name="output_password_key_vault_secret"></a> [password\_key\_vault\_secret](#output\_password\_key\_vault\_secret)

Description: Key Vault Secret resource that stores generated password.

### <a name="output_password_result"></a> [password\_result](#output\_password\_result)

Description: (String, Sensitive) The generated random string.

### <a name="output_rotation_rfc3339"></a> [rotation\_rfc3339](#output\_rotation\_rfc3339)

Description: (String) Configure the rotation timestamp with an [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339#section-5.8) format of the offset timestamp. When the current time has passed the rotation timestamp, the resource will trigger recreation. At least one of the 'rotation\_' arguments must be configured.

### <a name="output_time_rotating_id"></a> [time\_rotating\_id](#output\_time\_rotating\_id)

Description: (String) RFC3339 format of the `time_rotating`'s timestamp, e.g. `2020-02-12T06:36:13Z`. When the rotation occurs, this value will be updated to the new timestamp. This is useful for tracking when the resource was last rotated.

### <a name="output_tls_private_key_key_openssh"></a> [tls\_private\_key\_key\_openssh](#output\_tls\_private\_key\_key\_openssh)

Description: (String, Sensitive) Private key data in [OpenSSH PEM (RFC 4716)](https://datatracker.ietf.org/doc/html/rfc4716) format.

### <a name="output_tls_private_key_pem"></a> [tls\_private\_key\_pem](#output\_tls\_private\_key\_pem)

Description: (String, Sensitive) Private key data in [PEM (RFC 1421)](https://datatracker.ietf.org/doc/html/rfc1421) format.

### <a name="output_tls_private_key_pem_pkcs8"></a> [tls\_private\_key\_pem\_pkcs8](#output\_tls\_private\_key\_pem\_pkcs8)

Description: (String, Sensitive) Private key data in [PKCS#8 PEM (RFC 5208)](https://datatracker.ietf.org/doc/html/rfc5208) format.

### <a name="output_tls_private_key_public_key_fingerprint_md5"></a> [tls\_private\_key\_public\_key\_fingerprint\_md5](#output\_tls\_private\_key\_public\_key\_fingerprint\_md5)

Description: (String) The fingerprint of the public key data in OpenSSH MD5 hash format, e.g. `aa:bb:cc:...`. Only available if the selected private key format is compatible, similarly to `public_key_openssh` and the [ECDSA P224 limitations](../../docs#limitations).

### <a name="output_tls_private_key_public_key_fingerprint_sha256"></a> [tls\_private\_key\_public\_key\_fingerprint\_sha256](#output\_tls\_private\_key\_public\_key\_fingerprint\_sha256)

Description: (String) The fingerprint of the public key data in OpenSSH SHA256 hash format, e.g. `SHA256:...`. Only available if the selected private key format is compatible, similarly to `public_key_openssh` and the [ECDSA P224 limitations](../../docs#limitations).

### <a name="output_tls_private_key_public_key_openssh"></a> [tls\_private\_key\_public\_key\_openssh](#output\_tls\_private\_key\_public\_key\_openssh)

Description: (String) The public key data in ["Authorized Keys"](https://www.ssh.com/academy/ssh/authorized_keys/openssh#format-of-the-authorized-keys-file) format. This is not populated for `ECDSA` with curve `P224`, as it is [not supported](../../docs#limitations). **NOTE**: the [underlying](https://pkg.go.dev/encoding/pem#Encode) [libraries](https://pkg.go.dev/golang.org/x/crypto/ssh#MarshalAuthorizedKey) that generate this value append a `
` at the end of the PEM. In case this disrupts your use case, we recommend using [`trimspace()`](https://www.terraform.io/language/functions/trimspace).

### <a name="output_tls_private_key_public_key_pem"></a> [tls\_private\_key\_public\_key\_pem](#output\_tls\_private\_key\_public\_key\_pem)

Description: (String) Public key data in [PEM (RFC 1421)](https://datatracker.ietf.org/doc/html/rfc1421) format. **NOTE**: the [underlying](https://pkg.go.dev/encoding/pem#Encode) [libraries](https://pkg.go.dev/golang.org/x/crypto/ssh#MarshalAuthorizedKey) that generate this value append a `
` at the end of the PEM. In case this disrupts your use case, we recommend using [`trimspace()`](https://www.terraform.io/language/functions/trimspace).

### <a name="output_value_wo_version"></a> [value\_wo\_version](#output\_value\_wo\_version)

Description: (Number) Unix format of the `time_rotating`'s timestamp, e.g. `1581490573`. When the rotation occurs, this value will be updated to the new timestamp. This is useful for tracking when the resource was last rotated. You're encouraged to use this output as `value_wo_version` when you want to assign the ephemeral credential to write-only value.

## Modules

No modules.

<!-- markdownlint-disable-next-line MD041 -->
## Data Collection

The software may collect information about you and your use of the software and send it to Microsoft. Microsoft may use this information to provide services and improve our products and services. You may turn off the telemetry as described in the repository. There are also some features in the software that may enable you and Microsoft to collect data from users of your applications. If you use these features, you must comply with applicable law, including providing appropriate notices to users of your applications together with a copy of Microsoft’s privacy statement. Our privacy statement is located at <https://go.microsoft.com/fwlink/?LinkID=824704>. You can learn more about data collection and use in the help documentation and our privacy statement. Your use of the software operates as your consent to these practices.
<!-- END_TF_DOCS -->