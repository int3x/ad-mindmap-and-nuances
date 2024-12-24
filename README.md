# Active Directory mindmaps and nuances

A personal cheatsheet for enumerating and attacking Active Directory, plus a collection of gotchas.

## Mindmaps

- [Initial enumeration without credentials](<mindmaps/Initial.png>)

## Nuances

- [KRB_AP_ERR_SKEW or ntpdate issues](nuances/README.md#krb_ap_err_skew-or-ntpdate-issues-for-kerberos-authentication)
- [KDC_ERR_NEVER_VALID with faketime](nuances/README.md#kdc_err_never_valid-with-faketime)
- [KDC_ERR_S_PRINCIPAL_UNKNOWN](nuances/README.md#kdc_err_s_principal_unknown)
  - [KDC_ERR_S_PRINCIPAL_UNKNOWN on RBCD](nuances/README.md#kdc_err_s_principal_unknown-on-rbcd)
- [KDC_ERR_PADATA_TYPE_NOSUPP on certificate authentication](nuances/README.md#kdc_err_padata_type_nosupp-on-certificate-authentication)
- [DNS Resolution errors](nuances/README.md#dns-resolution-errors)
- [KDC_ERR_TGT_REVOKED](nuances/README.md#kdc_err_tgt_revoked)
  - [KDC_ERR_TGT_REVOKED on parent-child trust abuse](nuances/README.md#kdc_err_tgt_revoked-on-golden-or-trust-tickets)
