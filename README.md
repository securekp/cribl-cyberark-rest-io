----  
# CyberArk REST IO
----  
``` ```  

## About this Pack
This pack is designed to ingest audit logs from CyberArk Identity Security Platform (SaaS).  

It provides two REST collectors and related event breakers, as well as basic routes and pipelines for processing CyberArk SaaS audit data.

The pre-built pipelines will simply redact any fields with empty values, to provide a general starting point for event optimization. Further processing and/or enrichment may be desired for your use case(s).

``` ```  
``` ```  

## Deployment
``` ```  

---
### Create CyberArk Credentials
---
``` ```  

__Audit Logs/SIEM Integration Credentials:__

The CyberArk Audit Log Collector (cyberark_audit_logs) requires a CyberArk service account in order to properly query the CyberArk API for audit logs.

The following should help guide the creation of the required credentials:
- Create a service user (svc_cribl_audit) with Is OAuth confidential client; __note the full UPN__ (ex. svc_cribl_audit@cyberark.cloud.12345)
- Create and assign a role (cribl_audit_access) with Report Management under Administrative Rights
- Create an OAuth2 Server Custom Web app
  - Add the service user explicitly the Permissions tab
  - Create Scope with name cribl (this matters) and set Allowed REST APIs to .*
- Under Administration, create the SIEM integration and save the endpoints and API keys listed there

For more detailed guidance, refer to the CyberArk documentation @ [https://docs.cyberark.com/admin-space/latest/en/content/siem-integration/siem-export-3rd-party.htm](https://docs.cyberark.com/admin-space/latest/en/content/siem-integration/siem-export-3rd-party.htm)  

``` ```  
__Identity Security Platform Credentials:__

The CyberArk Identity Security Platform collector (cyberark_identity_logs) requires a CyberArk service account in order to properly query the Redrock API for Identity audit logs. 

The following should help guide the creation of the required credentials:
- Create a service user (svc_cribl_identity) with Is OAuth confidential client; __note the full UPN__ (ex. svc_cribl_identity@cyberark.cloud.12345)
- Create and assign a role (cribl_identity_access) with Report Management and Read Only System Administration under Administrative Rights
- Create an OAuth2 Client Custom Web app
  - Add the service user explicitly the Permissions tab
  - Create Scope with name cribl (this matters) and set Allowed REST APIs to .*

For more details guidance, refer to the CyberArk documentation @ [https://docs.cyberark.com/identity/latest/en/content/developer/use-queries.htm](https://docs.cyberark.com/identity/latest/en/content/developer/use-queries.htm)

``` ```  
``` ```  

---
### Configure Cribl Pack & Worker Group
---
``` ```  

After the configuration of CyberArk is complete, the following Cribl configuration is required.  

In the Worker Group where this pack is deployed, configure the following:

__Create Cribl Secrets for CyberArk Authentication:__
  * Navigate to __Worker Group Settings->Security->Secrets__ and create two Secrets as outlined below
  - __Name:__ cyberark_api_key
  - __Type__: API Key and Secret Key
  - __Values__: The API Key and Secret key values will be the credentials from the svc_cribl_audit
  * __Name:__ cyberark_audit_x_api_key
  * __Type__: Text
  * __Value__: API key from the SIEM integration

- __Notes:__
  - If you created separate API credentials for the Audit and Identity flows, then create a 3rd secret for the additional credentials and update the appropriate REST collector to reference the new Secret.
  - If you name your Secrets differently than we've provided above, make sure you also adjust the Secret references in the REST Collector(s) appropriately to align with your Secret names.
``` ```  

```------------------------------```  
``` ```  

Return to this pack and configure the following:

__Set Your CyberArk Hosts:__

  * Navigate to __Knowledge->Variables__
  * Adjust the __CyberArk_Hosts__ variable with the correct host values for your CyberArk instance:
      - __audit_host__ -- this will be the endpoint from the SIEM integration page
      - __identity_host__ -- this will be the tenant ID from your organization. (__Note__: Your tenant ID is NOT necessarily the friendly URL of your tenant)  

``` ```  
Configure one or both of the following collectors as-desired:

__Audit Logs Collector:__
  * Navigate to __Sources->REST->cyberark_audit_logs__
  - Discover:
    - Review the __Discover POST body__ query and make any desired changes to the pageSize, Selected Fields, or adding additional filters
    - Set the __x-api-key__ header field to reference the Secret using this JS:  __C.Secret("cyberark_audit_x_api_key", "text").value__
  * Collect:
    - Set the __x-api-key__ header field to reference the Secret using this JS:  __C.Secret("cyberark_audit_x_api_key", "text").value__
  - Authentication:
    - Set the __Client Secret value__ with the text: __client_credentials__
    - Set the __Authorization__ authentication header value to reference the Secret using this JS: __"Basic " + C.Encode.base64(C.Secret("cyberark_api_key", "keypair").apiKey + ":" + C.Secret("cyberark_api_key", "keypair").secretKey)__

``` ```  
__Identity Logs Collector:__
  * Navigate to __Sources->REST->cyberark_identity_logs__
  - Collect:
    - Review the __Collect POST body__ query and make any desired changes
    - Set the __X-IDAP-NATIVE-CLIENT__ collect header value as: __true__
  * Authentication Notes:
    - The _username_ and _password_ fields are not used so you can leave the default values here to satisfy the "required fields" UI validation
      - The actual credentials needed for the authentication are set within the __POST body__ field

``` ```  
__Note:__ The time based filters have been pre-configured in both collectors to allow for either the _Earliest/Latest_ run time values, or to use the full State Tracking, so no changes should generally be needed to those expressions.
* For more information on how the full State Tracking works, and how to set it up for a collector, check out the how-to videos posted in the [Cribl U YouTube](https://www.youtube.com/@Cribl_University/videos) channel!


``` ```  
``` ```  
__Final Steps:__
  * Configure your desired destination(s)
  - Review the routes and pipelines to complete the data flow and transform/enrich your events as-desired
  * Enable the __Scheduling__ for the configured collector(s)

``` ```  
``` ```  

---
``` ```  
``` ```  
## Upgrades
Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details.

``` ```  
``` ```  
## Release Notes
### Version 1.0.0 - 2026-02-09
Initial Release

### Version 0.9.3 - 2026-02-07
Internal beta

``` ```  
``` ```  
## Contributing to the Pack
To contribute to this Pack, or to report any issues or enhancement requests, please connect with Kelsey Prior (cribl.io) or Randy Corelli (cribl.io) on [Cribl Community Slack](https://cribl-community.slack.com).

``` ```  
``` ```  
## Contact
To contact us please email <kprior@cribl.io> or <rcorelli@cribl.io> or find us on the [Cribl Community Slack](https://cribl-community.slack.com).

``` ```  
``` ```  
## License
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).