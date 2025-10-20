# SailPoint IdentityIQ (IIQ) Feature Configuration Guide

## Introduction

This document provides a starting base for configuring all major SailPoint IdentityIQ (IIQ) features in a test lab environment integrated with FreeIPA via the LDAP connector. It assumes:
- IIQ (version 8.1 or later) is installed and accessible (e.g., `http://localhost:8080/identityiq`).
- FreeIPA is set up as an application in IIQ with LDAP connector, aggregating users and groups.
- Admin access to IIQ for configuration.

The guide covers all core IIQ features: Lifecycle Manager, Compliance Manager, Password Management, Access Risk Management, and additional features like SSO, auditing, and reporting. Each section includes configuration steps, placeholder for screenshots (to be added later), and test instructions. Customize this base as needed for your lab.

**Note**: Refer to [SailPoint Documentation](https://documentation.sailpoint.com/identityiq/help/) for version-specific details.

## 1. Lifecycle Manager

Lifecycle Manager handles access requests, provisioning, and lifecycle events (e.g., joiner/mover/leaver).

### Configuration
1. **Enable Lifecycle Manager**:
   - Navigate to **Gear Icon > Global Settings > IdentityIQ Configuration > Lifecycle Manager**.
   - Check **Enable Lifecycle Manager**.
2. **Configure Access Requests**:
   - Go to **Setup > Request Access**.
   - Add roles/entitlements: Select FreeIPA groups (e.g., `cn=admins,cn=groups,dc=example,dc=com`) as requestable items.
   - Set **Approval Workflow**: Use default "Access Request Approval" or customize in **Setup > Workflows**.
     - Example: Manager approval → Group owner approval.
   - Enable **Self-Service Access Requests** in Global Settings.
3. **Define Lifecycle Events**:
   - **Setup > Lifecycle Events > New Event**.
   - Example: "Joiner" event:
     - Trigger: New identity detected via aggregation.
     - Action: Assign default role (e.g., "Employee").
   - Example: "Leaver" event:
     - Trigger: Account disabled in FreeIPA.
     - Action: Disable IIQ identity, revoke entitlements.
4. **Provisioning Policies**:
   - In FreeIPA app config (**Setup > Applications > FreeIPA_LDAP**), go to **Provisioning** tab.
   - Add policies:
     - **Create**: Map IIQ attributes (e.g., `firstName` → `givenName`, `uid` → `uid`).
     - **Update/Delete**: Similar mappings.
   - Use forms for UI: **Setup > Forms > Provisioning Policy - Identity**.
5. **Schedule Tasks**:
   - **Setup > Tasks > New Task > Lifecycle Event Refresh**.
   - Schedule daily to process events.

### Screenshot Placeholder
- [Add screenshot: Lifecycle Manager settings page]
- [Add screenshot: Access request workflow configuration]

### Test
- Submit an access request for a FreeIPA group via **My Access > Request Access**.
- Approve via **Approvals > Pending Requests**.
- Verify provisioning in FreeIPA: `ipa user-show testuser`.

## 2. Compliance Manager

Compliance Manager supports certifications, policy enforcement, and Separation of Duties (SoD).

### Configuration
1. **Enable Compliance Manager**:
   - **Gear Icon > Global Settings > IdentityIQ Configuration > Compliance**.
   - Check **Enable Certifications** and **Enable Policy Enforcement**.
2. **Certifications**:
   - **Setup > Certifications > New Campaign**.
   - Types:
     - **Identity Certification**: Review all FreeIPA accounts.
     - **Entitlement Owner**: Review group memberships (e.g., `admins` group).
     - **Role Composition**: Validate role assignments.
   - Set schedule: Quarterly, starting [date].
   - Assign certifiers: Managers or group owners.
3. **Separation of Duties (SoD)**:
   - **Setup > Policies > New Policy > Separation of Duties**.
   - Example: Users cannot have both `admins` and `auditors` FreeIPA groups.
     - Rule: `if identity.hasEntitlement("cn=admins,...") && identity.hasEntitlement("cn=auditors,...") then flag violation`.
   - Enforcement: Block provisioning or flag in certifications.
4. **Policy Violations**:
   - **Setup > Policies > New Policy > Identity Policy**.
   - Example: Flag accounts with expired FreeIPA passwords (`krbPasswordExpiration`).
   - Schedule scans: **Tasks > New Task > Policy Scan**.

### Screenshot Placeholder
- [Add screenshot: Certification campaign setup]
- [Add screenshot: SoD policy rule]

### Test
- Launch a certification campaign.
- Check certifier’s inbox (**Certifications > [Campaign Name]**).
- Simulate SoD violation by assigning conflicting groups; verify alert.

## 3. Password Management

Password Management enables self-service password resets and synchronization.

### Configuration
1. **Enable Password Management**:
   - **Gear Icon > Global Settings > Password Management**.
   - Check **Enable Password Management**.
2. **Configure FreeIPA as Managed App**:
   - In FreeIPA app (**Setup > Applications > FreeIPA_LDAP**), enable **Password Management**.
   - Map attributes: `userPassword` for FreeIPA.
3. **Password Policy**:
   - **Setup > Password Policies > New Policy**.
   - Settings: Min length 8, require special characters, expire every 90 days.
   - Apply to FreeIPA app.
4. **Self-Service Reset**:
   - **Global Settings > Login > Password Reset**.
   - Enable **Forgot Password** with email/SMS challenge.
   - Configure questions: **Setup > Authentication Questions**.

### Screenshot Placeholder
- [Add screenshot: Password policy settings]
- [Add screenshot: Self-service reset page]

### Test
- Log in as a test user, use **Forgot Password** link.
- Reset password and verify in FreeIPA: `ipa user-show testuser`.

## 4. Access Risk Management

Access Risk Management identifies and mitigates risky access.

### Configuration
1. **Enable Access Risk**:
   - **Gear Icon > Global Settings > Access Risk**.
   - Check **Enable Access Risk**.
2. **Define Risk Scores**:
   - **Setup > Access Risk > Risk Scoring**.
   - Example: Assign high risk (score 80) to FreeIPA `admins` group.
   - Composite score: Combine entitlement risk + identity attributes (e.g., department).
3. **Risk Rules**:
   - **Setup > Access Risk > Rules**.
   - Example: Flag identities with `uidNumber < 1000` (system accounts).
4. **Integration with Certifications**:
   - Link high-risk entitlements to certification campaigns for priority review.

### Screenshot Placeholder
- [Add screenshot: Risk scoring configuration]
- [Add screenshot: Risk rule editor]

### Test
- Assign `admins` group to a test user.
- Check risk score in **Identities > [User] > Risk**.
- Verify high-risk items appear in certifications.

## 5. Single Sign-On (SSO)

SSO integrates IIQ with FreeIPA’s Kerberos for seamless authentication.

### Configuration
1. **Enable SSO**:
   - **Gear Icon > Global Settings > Login > SSO**.
   - Select **SAML** or **Kerberos** (FreeIPA supports Kerberos).
2. **Kerberos Setup**:
   - Configure FreeIPA as IdP: `ipa service-add HTTP/iiq.example.com`.
   - Export keytab: `ipa-getkeytab -s freeipa.example.com -p HTTP/iiq.example.com -k /path/to/iiq.keytab`.
   - In IIQ, **Global Settings > SSO > Kerberos**:
     - Keytab path: `/path/to/iiq.keytab`.
     - Realm: `EXAMPLE.COM`.
3. **Test SSO**:
   - Configure browser for Kerberos (e.g., Firefox SPNEGO).
   - Enable in **Login > SSO Options**.

### Screenshot Placeholder
- [Add screenshot: SSO configuration page]
- [Add screenshot: Kerberos keytab setup]

### Test
- Access IIQ URL (`http://iiq.example.com:8080/identityiq`).
- Verify automatic login with FreeIPA credentials.

## 6. Auditing

Auditing tracks all IIQ actions for compliance.

### Configuration
1. **Enable Auditing**:
   - **Gear Icon > Global Settings > Audit Configuration**.
   - Check **Enable Auditing**.
   - Select events: Access requests, provisioning, certifications, logins.
2. **Audit Rules**:
   - **Setup > Audit Rules**.
   - Example: Log all provisioning to FreeIPA `admins` group.
3. **Retention**:
   - Set log retention: 90 days (adjust in **Global Settings > Audit**).

### Screenshot Placeholder
- [Add screenshot: Audit event selection]
- [Add screenshot: Audit log viewer]

### Test
- Perform an access request and provisioning.
- Check **Monitor > Audit > Audit Log** for entries.

## 7. Reporting

Reporting provides insights into identities, access, and compliance.

### Configuration
1. **Enable Reporting**:
   - **Gear Icon > Global Settings > Reports**.
   - Check **Enable Reporting**.
2. **Create Reports**:
   - **Analytics > Reports > New Report**.
   - Examples:
     - **Identity Report**: List all FreeIPA users with attributes (`uid`, `mail`).
     - **Entitlement Report**: Show group memberships.
     - **Certification Status**: Track campaign progress.
   - Format: Table, export as CSV/PDF.
3. **Schedule Reports**:
   - **Setup > Tasks > New Task > Report Task**.
   - Schedule weekly email delivery to admins.

### Screenshot Placeholder
- [Add screenshot: Report creation interface]
- [Add screenshot: Sample report output]

### Test
- Run an identity report.
- Export and verify FreeIPA user data.

## 8. Full-Text Search

Full-text search enables quick lookups of identities and objects.

### Configuration
1. **Enable Search**:
   - **Gear Icon > Global Settings > Search**.
   - Check **Enable Full-Text Search** (uses Lucene).
2. **Index Configuration**:
   - **Setup > Search Configuration**.
   - Index attributes: `uid`, `cn`, `mail`, group names.
3. **Search Scope**:
   - Include identities, accounts, entitlements.

### Screenshot Placeholder
- [Add screenshot: Search configuration page]
- [Add screenshot: Search results UI]

### Test
- Search for a FreeIPA user (e.g., `uid=testuser`) in **Search** bar.
- Verify results show identity and group details.

## 9. Attachments

Attachments support documentation for requests and certifications.

### Configuration
1. **Enable Attachments**:
   - **Gear Icon > Global Settings > Miscellaneous**.
   - Check **Enable Attachments**.
2. **Configure Limits**:
   - Max file size: 5MB (adjust in **Global Settings**).
   - Allowed types: PDF, PNG, JPEG.
3. **Integration**:
   - Enable for access requests and certifications.

### Screenshot Placeholder
- [Add screenshot: Attachment upload UI]
- [Add screenshot: Attachment in request details]

### Test
- Submit an access request with a PDF attachment.
- Verify attachment in **Approvals > Request Details**.

## 10. Role Management

Role Management organizes access into business-friendly roles.

### Configuration
1. **Enable Role Management**:
   - **Gear Icon > Global Settings > Role Management**.
   - Check **Enable Role Management**.
2. **Define Roles**:
   - **Setup > Roles > New Role**.
   - Example:
     - **Business Role**: "IT Admin".
     - Entitlements: FreeIPA `admins` group.
     - Inheritance: Include "Employee" role.
3. **Role Mining**:
   - **Analytics > Role Mining**.
   - Analyze FreeIPA group patterns to suggest roles.
4. **Role Assignment**:
   - Auto-assign via lifecycle events or manual requests.

### Screenshot Placeholder
- [Add screenshot: Role creation interface]
- [Add screenshot: Role mining results]

### Test
- Create an "IT Admin" role.
- Assign to a test user via request or lifecycle event.
- Verify FreeIPA group membership.

## Troubleshooting

- **Lifecycle Issues**: Check workflows in **Debug > Workflows**. Ensure provisioning policies match FreeIPA schema.
- **Certification Errors**: Verify campaign filters include FreeIPA accounts.
- **Password Sync Fails**: Confirm `userPassword` attribute is writable in FreeIPA.
- **SSO Problems**: Validate keytab and Kerberos realm in FreeIPA.
- **Logs**: Enable debug logging in `log4j2.xml` (`/opt/tomcat/webapps/identityiq/WEB-INF/classes`).
- **General**: Use **Debug Pages** (`http://iiq.example.com:8080/identityiq/debug`) for object inspection.

## Next Steps

1. Add screenshots to placeholders for visual reference.
2. Test each feature with a small FreeIPA dataset (e.g., 10 users).
3. Expand configurations in `configs/freeipa.properties` for advanced settings.
4. Automate test data creation via `scripts/create_freeipa_users.sh`.
5. Schedule tasks for ongoing operations (aggregation, certifications, reports).
6. Explore advanced features (e.g., PAM, AI-driven analytics) if licensed.

This document is a starting point. Customize based on your lab's requirements and add screenshots to enhance clarity.
