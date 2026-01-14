# CyberArk REST IO
----

This pack is designed to ingest audit logs from CyberArk Identity Security Platform (SaaS).

## About this Pack
This pack provides a REST collector, event breaker, and a basic route and pipeline for parsing CyberArk audit data from their SaaS platform.

## Deployment
This pack requires a service account in CyberArk to be configured in order to properly query the Redrock API for audit logs. The following should help guide the creation of this: 
- Create a service user (svc_cribl_audit) with Is OAuth confidential client; note the full UPN (ex. svc_cribl_audit@cyberark.cloud.12345)
- Create and assign a role (Cribl_Audit_Access) with Read Only System Administrator under Administrative Rights
- Create an OAuth2 Client Custom Web app
  - Add the service user explicitly the Permissions tab
  - Create Scope with name cribl (this matters) and set Allowed REST APIs to .*

After the configuration of CyberArk is complete, return to this pack and do the following: 
- In the Worker Group where this pack will be deployed, create a Secret called CyberArk_API_Key with the credentials from the above configuration.
- Navigate to the REST collector in this pack and update the collect and login URLs to indicate your tenant.

## Upgrades
Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details.

## Release Notes

### Version 0.9.0 - 2026-01-14

Internal beta

## Contributing to the Pack
To contribute to this Pack, or to report any issues or enhancement requests, please connect with Kelsey Prior (cribl.io) on [Cribl Community Slack](https://cribl-community.slack.com).

## Contact
To contact us please email <kprior@example.com>.

## License
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).