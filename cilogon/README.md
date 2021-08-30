# CILogon

## Related

See also

- [Workshop](../workshop) notes for ongoing AAI discussion
- User Guide [Login & Signup](../user-guide/login-signup.md) section

## Configure CILogon with Gen3 Fence

> The support for CILogon in the Fence service was implemented by [Scott Koranda](https://github.com/skoranda) with Pull Request [#896](https://github.com/uc-cdis/fence/pull/896).

Update the Cloud Automation deployment manifest to use `fence:2021.08` or recent release.

```bash
# update fence version (if needed)
vim $HOME/cdis-manifest/gen3.cloud.dev.umccr.org/manifest.json

  "fence": "quay.io/cdis/fence:2021.08"
```

Enable CILogon in the Fence config. An example can be found in the [config-default.yaml](https://github.com/uc-cdis/fence/blob/master/fence/config-default.yaml).
```bash
vim $HOME/Gen3Secrets/apis_configs/fence-config.yaml
```

To get the `client_id` and `client_secret` you create a new `OIDC Client` in your COmanage Registry account (`Configuration` menu). Don't forget to add LDAP to Claim mapping that maps LDAP Attribute Name `voPersonApplicationUID;app-gen3` to OIDC Claim Name `sub` to get a readable and consistent username (even if you link multiple identities to your COmanage account).

## COmanage Registry

Update COmanage to add an OIDC Client to the Collaborative Organisation (CO) that will access Gen3. The OIDC client provides the ‘client_idandclient_secret` for the Fence config.

The CILogon OIDC config expects a Gen3 username. The following steps provide a Gen3 username to the CILogon configuration and enable storage of the corresponding value in the CILogon LDAP repository.

From with the CO, add an Extended Type for the Gen3 username (Configuration > Extended Types).
[[imgs/extended types.jpg]]

Add an Identifier Assignment for a GEN3Username (Configuration > Identifier Assignments).
[[./imgs/identifier assignment.jpg]]

Add a Service for Gen3 (Configuration > Services).
[[./imgs/services.jpg]]

Add the OIDC CLient for Gen3 (Configuration > OIDC Clients).
[[./imgs/OIDC client.jpg]]

Minting a new ODIC Claim for Gen3 necessitates that CILogon.org has boarded the CO in order to delegate the management of OIDC clients to the CO. This enables the CO admin or COmanage platform admin to mint new OIDC clients without further approval.

Once the CO has the ability to mint OIDC Clients, click Add a New OIDC Client, fill the fields and submit the form. The client ID and secret will be automatically generated. Add the secret to Fence configuration - CILogon only records a secure hash of the value and cannot recover the secret if it is lost. If that happens, just create a new client and reconfigure the service.

To retrieve a user’s Gen3Username from LDAP, there should be a LDAP to Claim mapping for voPersonApplicationUID;app-gen3 to OIDC Claim Name sub.
[[./imgs/OIDC client.jpg]]
