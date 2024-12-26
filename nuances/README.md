# Nuances for AD enumeration and exploitation

A list of gotchas and nuances/subtleties that I've encountered while attacking Active Directory misconfigurations.

## Index

- [KRB_AP_ERR_SKEW or ntpdate issues](#krb_ap_err_skew-or-ntpdate-issues-for-kerberos-authentication)
- [KDC_ERR_NEVER_VALID with faketime](#kdc_err_never_valid-with-faketime)
- [KDC_ERR_S_PRINCIPAL_UNKNOWN](#kdc_err_s_principal_unknown)
  - [KDC_ERR_S_PRINCIPAL_UNKNOWN on RBCD](#kdc_err_s_principal_unknown-on-rbcd)
- [STATUS_MORE_PROCESSING_REQUIRED](#status_more_processing_required)
- [KDC_ERR_PADATA_TYPE_NOSUPP on certificate authentication](#kdc_err_padata_type_nosupp-on-certificate-authentication)
- [DNS Resolution errors](#dns-resolution-errors)
- [KDC_ERR_TGT_REVOKED](#kdc_err_tgt_revoked)
  - [KDC_ERR_TGT_REVOKED on golden or trust tickets](#kdc_err_tgt_revoked-on-golden-or-trust-tickets)
- [INSUFF_ACCESS_RIGHTS](#insuff_access_rights)
  - [INSUFF_ACCESS_RIGHTS while updating msDS-KeyCredentialLink](#insuff_access_rights-while-updating-msds-keycredentiallink)
- [STATUS_ACCESS_DENIED](#status_access_denied)
  - [STATUS_ACCESS_DENIED while adding members to group](#status_access_denied-while-adding-members-to-group)

## KRB_AP_ERR_SKEW or ntpdate issues for kerberos authentication

For the pre-authentication step of kerberos authentication, the system time on the attacker machine needs to be synced with the time on target machine.  
The error `KRB_AP_ERR_SKEW` shows up otherwise.  
`ntpdate` is a linux utility which synchronizes the system time to that of a target NTP server.

```bash
sudo ntpdate target.tld
```

However, virtualization solutions like VMware and VirtualBox have a feature to force sync the time on Virtual Machines with the time on host machine.  
This feature is not desirable for kerberos authentication. On VMWare Workstation, it can be disabled by unchecking `Synchronize guest time with host` in `Virtual Machine Settings > Options > VMware Tools`

## KDC_ERR_NEVER_VALID with faketime

`faketime` is an alternative to `ntpdate` for time synchronization. Unlike `ntpdate` which changes time system-wide, `faketime` changes time for a given command.  
It intercepts syscalls which retrieve time, and reports "faked" times.

```bash
faketime "$(ntpdate -q target.tld | cut -d ' ' -f 1,2)" getTGT.py target.tld/domainuser:password
```

Sometimes, only a fraction of the time-retrieval syscalls used by a tool get intercepted, and that results in the `KDC_ERR_NEVER_VALID` error.  
It implies that the expiry time for a TGT is before its starting time. That can happen when the faked time is used as starting time, but expiry time gets calculated using real time.  
The solution to such issues is to change the system-wide time with `ntpdate`.

## KDC_ERR_S_PRINCIPAL_UNKNOWN

### KDC_ERR_S_PRINCIPAL_UNKNOWN on RBCD

When performing Resource-Based Constrained Delegation (RBCD) attack, sometimes the error `KDC_ERR_S_PRINCIPAL_UNKNOWN` is encountered while using `getST.py`  
It could suggest that the impersonating account does not have an SPN set, nor is a machine account.  
[SPN-less RBCD](https://www.tiraniddo.dev/2022/05/exploiting-rbcd-using-normal-user.html) can still be invoked. However, the steps are more involved, and it renders the account unusable.

## STATUS_MORE_PROCESSING_REQUIRED

I'm uncertain about this error. It shows up randomly and sometimes goes away when the same command is used again.

## KDC_ERR_PADATA_TYPE_NOSUPP on certificate authentication

When performing certificate authentication with tools like `certipy`, sometimes the error `KDC_ERR_PADATA_TYPE_NOSUPP` is encountered.  
It can be caused due to several reasons, but I know of two: (a) when the root CA certificate expires and (b) when the Domain Controllers have `Smart Card Logon` or `Server Authentication` EKU missing from the certificates.  
I'm certain that poorly configured PKINIT can also result in this error.  
Sometimes, [PassTheCert](https://github.com/AlmondOffSec/PassTheCert/tree/main/Python) or tools with `ldap-shell` baked-in can be used as workaround.

## DNS Resolution errors

DNS resolution errors are also frequent with Active Directory pentesting tools.  
In some cases, [DNSChef](https://github.com/iphelix/dnschef) is a good workaround for such errors.  
It starts a DNS resolver locally and all records can be controlled or proxied:

```bash
sudo python3 dnschef.py --fakeip 10.10.10.10 --fakedomains target.tld
```

We can then point other tools to use 127.0.0.1 as nameserver.

## KDC_ERR_TGT_REVOKED

### KDC_ERR_TGT_REVOKED on golden or trust tickets

When using `ticketer.py` to forge golden or trust tickets, the error `KDC_ERR_TGT_REVOKED` can be encountered sometimes.  
The error can be caused due to the patches made against `noPAC`. After the October 2022 patches, the KDC expects the PAC to contain a `PAC_REQUESTOR` structure.  
As a result, a valid `user-id` and `username` needs to be specified to `ticketer.py`. Furthermore, the AESkey needs to be used instead of the NThash.

```bash
ticketer.py -aesKey ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff -domain-sid S-1-5-21-0000000000-0000000000-111111111 -domain child.target.tld -extra-sid S-1-5-21-0000000000-000000000-222222222-519 -user-id 1111 'existing_user'
```

## INSUFF_ACCESS_RIGHTS

### INSUFF_ACCESS_RIGHTS while updating msDS-KeyCredentialLink

When performing shadow credentials attack, i.e. updating msDS-KeyCredentialLink attribute, the `INSUFF_ACCESS_RIGHTS` error can show up if:

- Privileges are legitimately lacking.
- Only machine accounts can edit their own `msDS-KeyCredentialLink`, user accounts cannot.
- The `msDS-KeyCredentialLink` is already populated. In that case, it needs to be cleared first.

## STATUS_ACCESS_DENIED

### STATUS_ACCESS_DENIED while adding members to group

After abusing WriteOwner privilege over a group, the error `STATUS_ACCESS_DENIED` can be encountered while trying to add members.  
Even though the group owner has WriteDacl privilege over the group, the AddMember privilege is absent in the DACL ACEs.  
The WriteDacl needs to be utilized to add the required ACEs:

```bash
owneredit.py -action write -new-owner jdoe -target-dn 'CN=TargetGroup,CN=Users,DC=target,DC=tld' target.tld/jdoe:password
dacledit.py -action write -rights FullControl -principal jdoe -target-dn 'CN=TargetGroup,CN=Users,DC=target,DC=tld' -inheritance target.tld/jdoe:password
```

After using `dacledit.py`, members can be added without errors.
