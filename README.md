# CyberArk REST IO
----

This pack is designed to ingest audit logs from CyberArk Identity Security Platform (SaaS).

## About this Pack
This pack provides a couple of REST collectors, event breakers, and a basic routes and pipelines for parsing CyberArk audit data from their SaaS platform.

## Deployment
For the CyberArk Audit Log Collector (cyberark_audit_logs):
This source requires a service account in CyberArk to be configured in order to properly query the CyberArk API for audit logs. The following should help guide the creation of this (Refer to https://docs.cyberark.com/admin-space/latest/en/content/siem-integration/siem-export-3rd-party.htm for detailed CyberArk setup): 
- Create a service user (svc_cribl_audit) with Is OAuth confidential client; note the full UPN (ex. svc_cribl_audit@cyberark.cloud.12345)
- Create and assign a role (Cribl_Audit_Access) with Report Management under Administrative Rights
- Create an OAuth2 Server Custom Web app
  - Add the service user explicitly the Permissions tab
  - Create Scope with name cribl (this matters) and set Allowed REST APIs to .*


For the CyberArk Identity Security Platform Source (cyberark_identity_logs):
This source requires a service account in CyberArk to be configured in order to properly query the Redrock API for Identity audit logs. The following should help guide the creation of this (Refer to https://docs.cyberark.com/identity/latest/en/content/developer/use-queries.htm for detailed CyberArk setup): 
- Create a service user (svc_cribl_identity) with Is OAuth confidential client; note the full UPN (ex. svc_cribl_identity@cyberark.cloud.12345)
- Create and assign a role (Cribl_Identity_Access) with Report Management under Administrative Rights
- Create an OAuth2 Client Custom Web app
  - Add the service user explicitly the Permissions tab
  - Create Scope with name cribl (this matters) and set Allowed REST APIs to .*

After the configuration of CyberArk is complete, return to this pack and do the following: 
- In the Worker Group where this pack will be deployed, create a Secret called CyberArk_API_Key with the credentials from the above configuration. Note: The username and password fields in the REST collector are not used, so there's just nonsense in there. The Secret is leveraged within the Collect POST body. 
- Navigate to the REST collectors in this pack and update the collect and login URLs to indicate your tenant ID. Note: Your tenant ID is NOT necessarily the friendly URL of your tenant. 
- Configure the destinations you want to send the data to, and then configure the routes and attach the pipelines.

## Upgrades
Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details.

## Release Notes

### Version 0.9.1 - 2026-01-16

Internal beta

## Contributing to the Pack
To contribute to this Pack, or to report any issues or enhancement requests, please connect with Kelsey Prior (cribl.io) on [Cribl Community Slack](https://cribl-community.slack.com).

## Contact
To contact us please email <kprior@cribl.io>.

## License
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).