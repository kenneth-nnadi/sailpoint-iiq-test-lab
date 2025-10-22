# SailPoint IdentityIQ (IIQ) Feature Configuration Guide

## Introduction

This document provides a starting base for configuring all major SailPoint IdentityIQ (IIQ) features in a test lab environment integrated with FreeIPA via the LDAP connector. It assumes:
- IIQ (version 8.3 or later) is installed and accessible (e.g., `http://localhost:8080/identityiq`).
- FreeIPA is set up as an application in IIQ with LDAP connector, aggregating users and groups.([Check the Installation steps here](./freeipa_deployment.md))
- Admin access to IIQ for configuration. ([Check the Installation steps and Login here](./environment_setup.md))

The guide covers all core IIQ features: Lifecycle Manager, Compliance Manager, Password Management, Access Risk Management, and additional features like SSO, auditing, and reporting. Each section includes configuration steps,and pictures.

**Note**: Refer to [SailPoint Documentation](https://documentation.sailpoint.com/identityiq/help/) for version-specific details.

## 1. Application Definition

The Application Definition configures FreeIPA as a managed system in IIQ, enabling connectivity and schema mapping for users and groups via the LDAP connector.

### Configuration
1. **Access Application Definition**:
   - Log in to IIQ as an admin.
   - Navigate to **Setup > Applications > Application Definition**.
   - Click **+ New Application** to start.

2. **Basic Settings**:
   - **Name**: `FreeIPA_LDAP` (descriptive, unique name).
   - **Type**: Select **LDAP** from the connector dropdown.
   - **Owner**: Assign an admin user (e.g., `admin`).
   - **Description**: "FreeIPA LDAP integration for identity governance."
  
     <img width="1373" height="788" alt="image" src="https://github.com/user-attachments/assets/05ad3ad9-f99f-479a-a3e7-756e83f2ae04" />


3. **Connection Settings**:
   - Configure the LDAP connector to connect to FreeIPA:
     | Field                  | Value/Example                          | Description |
     |------------------------|----------------------------------------|-------------|
     | **Host**              | `freeipa.example.com`                 | FreeIPA server hostname or IP. |
     | **Port**              | `636`                                 | Use 389 for non-SSL (not recommended). |
     | **Base DN**           | `dc=example,dc=com`                   | Root suffix for LDAP searches. |
     | **Bind DN**           | `uid=iiq-service,cn=users,cn=accounts,dc=example,dc=com` | Service account DN for authentication. |
     | **Bind Credentials**  | `[Enter password]`                    | Password for the service account. |
     | **Authentication Type**| `simple`                              | Use `GSSAPI` for Kerberos if configured. |
     | **Use SSL**           | Enabled                               | Check for LDAPS (recommended). |
     | **Connection Timeout**| `30`                                  | Seconds for connection timeout. |
     | **Authentication Search Attributes**      | `uid`                             | Specifies the user attribute used to identify and authenticate accounts.
     

<img width="1370" height="784" alt="image" src="https://github.com/user-attachments/assets/8fdf6499-d8be-4c25-9e80-9a1c02cf6a19" />


4. **Schema Configuration**:
   - Go to the **Schema** tab in the application config.
   - **Account Schema** (for FreeIPA users):
     - **ObjectClass**: `posixAccount`, `inetOrgPerson`, `ipaUser`.
     - **Attributes**:
       | Attribute Name | LDAP Attribute | Type   | Description |
       |----------------|----------------|--------|-------------|
       | `nativeIdentity` | `uid`         | String | Unique user ID (e.g., `testuser`). |
       | `displayName`  | `cn`           | String | Full name (e.g., `Test User`). |
       | `firstName`    | `givenName`    | String | First name. |
       | `lastName`     | `sn`           | String | Last name. |
       | `email`        | `mail`         | String | Email address. |
       | `uidNumber`    | `uidNumber`    | Integer | POSIX user ID. |
       | `groups`       | `memberOf`     | Multi-Valued String | Groups user belongs to (DN format). |
     - Set `uid` as the **Identity Attribute** for correlation.
   - **Group Schema** (for FreeIPA groups/entitlements):
     - **ObjectClass**: `posixGroup`, `groupOfNames`, `ipaUserGroup`.
     - **Attributes**:
       | Attribute Name | LDAP Attribute | Type   | Description |
       |----------------|----------------|--------|-------------|
       | `name`         | `cn`           | String | Group name (e.g., `admins`). |
       | `gidNumber`    | `gidNumber`    | Integer | POSIX group ID. |
       | `members`      | `memberUid`    | Multi-Valued String | List of user UIDs in the group. |
     - Enable **Multi-Valued Groups** to handle multiple group memberships.
   - **Filters**:
     - Accounts: `(objectClass=posixAccount)` to fetch users.
     - Groups: `(objectClass=posixGroup)` to fetch groups.

<img width="1367" height="784" alt="image" src="https://github.com/user-attachments/assets/35a98343-ceac-4026-b354-17a8600018ec" />


5. **Correlation Configuration**:
   - Go to **Correlation** tab.
   - Map FreeIPA `uid` to IIQ identity attribute (e.g., `employeeId` or `name`).
   - Example: `application.uid = identity.employeeId`.

6. **Provisioning Settings**:
   - Go to **Provisioning** tab.
   - Enable **Create**, **Update**, **Delete**, **Enable/Disable**, and **Unlock** operations.
   - Create provisioning policy:
     - **Create Policy**:
       | Field       | Value/Example       | Description |
       |-------------|---------------------|-------------|
       | `uid`       | `identity.name`     | Generate UID from IIQ identity name. |
       | `givenName` | `identity.firstName`| Map first name. |
       | `sn`        | `identity.lastName` | Map last name. |
       | `cn`        | `identity.displayName` | Full name. |
       | `mail`      | `identity.email`    | Email address. |
       | `uidNumber` | `[Auto-generate]`   | Let FreeIPA assign POSIX UID. |
     - **Update/Delete**: Similar mappings, with conditions for modification.

7. **Test Connection**:
   - Click **Test Connection** in the application config.
   - Verify connectivity to FreeIPA (check logs in `/opt/tomcat/logs/iiq.log` if it fails).
     
     <img width="1369" height="787" alt="image" src="https://github.com/user-attachments/assets/30c2dfb0-3bd6-4e7d-a074-11e60c5f6d54" />


8. **Save Application**:
   - Click **Save** to store the configuration.

### Test
- Go to **Debug > Applications > FreeIPA_LDAP**.
- Verify application object details (connection, schema).
- Run a quick LDAP query from IIQ server to confirm:
  ```bash
  ldapsearch -x -H ldaps://freeipa.example.com -D "uid=iiq-service,..." -W -b "dc=example,dc=com" "(uid=testuser)"
  ```
- Check **Debug > Logs** for connection errors.

## 2. Account Aggregation

Account Aggregation imports FreeIPA users and groups into IIQ to populate identities and entitlements.

### Configuration
1. **Create Account Aggregation Task**:
   - Navigate to **Setup > Tasks > New Task**.
   - Select **Account Aggregation**.
   - **Settings**:
     | Field            | Value/Example              | Description |
     |------------------|----------------------------|-------------|
     | **Name**         | `FreeIPA_Account_Aggregation` | Descriptive task name. |
     | **Application**  | `FreeIPA_LDAP`            | Select the FreeIPA app. |
     | **Object Type**  | `account`                 | Import user accounts. |
     | **Filter**       | `(objectClass=posixAccount)` | LDAP filter for users. |
     | **Incremental**  | Disabled                  | Full sync for initial setup. |
     | **Max Records**  | `1000`                    | Limit for testing (adjust for prod). |
     | **Promote Attributes** | `uid, cn, mail, memberOf` | Attributes to store in IIQ. |

2. **Create Group Aggregation Task**:
   - **Setup > Tasks > New Task > Account Aggregation**.
   - **Settings**:
     | Field            | Value/Example              | Description |
     |------------------|----------------------------|-------------|
     | **Name**         | `FreeIPA_Group_Aggregation` | Descriptive task name. |
     | **Application**  | `FreeIPA_LDAP`            | Select the FreeIPA app. |
     | **Object Type**  | `group`                   | Import group entitlements. |
     | **Filter**       | `(objectClass=posixGroup)` | LDAP filter for groups. |
     | **Incremental**  | Disabled                  | Full sync for initial setup. |
     | **Max Records**  | `500`                     | Limit for testing. |
     | **Promote Attributes** | `cn, gidNumber, memberUid` | Attributes to store in IIQ. |

3. **Correlation Rule**:
   - **Setup > Identity Configuration > Correlation**.
   - Example rule: Map FreeIPA `uid` to IIQ `employeeId` or `name`.
   - Create rule in **Setup > Rules**:
     ```xml
     <Rule name="FreeIPA_Correlation" type="Correlation">
       <Source>
         <![CDATA[
           import sailpoint.object.*;
           return identity.getAttribute("employeeId") == account.getAttribute("uid");
         ]]>
       </Source>
     </Rule>
     ```

4. **Schedule Tasks**:
   - Go to **Setup > Scheduler**.
   - Add tasks to run daily:
     - **FreeIPA_Account_Aggregation**: 2 AM.
     - **FreeIPA_Group_Aggregation**: 3 AM.
   - Enable **Automatic Execution**.

5. **Identity Refresh**:
   - After aggregation, run **Identity Refresh** to correlate accounts to identities:
     - **Setup > Tasks > New Task > Identity Refresh**.
     - Select **Full Refresh**.
     - Enable **Correlate Accounts** and **Refresh Entitlements**.

### Screenshot Placeholder
- [Add screenshot: Account Aggregation task setup]
- [Add screenshot: Group Aggregation task configuration]
- [Add screenshot: Identity Refresh task results]

### Test
- Run **FreeIPA_Account_Aggregation** task (**Monitor > Task Results**).
- Verify imported accounts in **Identities > [User] > Accounts**.
- Run **FreeIPA_Group_Aggregation** task.
- Check entitlements in **Applications > FreeIPA_LDAP > Entitlements**.
- Run **Identity Refresh** and confirm identities link to FreeIPA accounts (e.g., `uid=testuser` maps to an IIQ identity).
- Check logs (`/opt/tomcat/logs/iiq.log`) for errors.

## Troubleshooting

- **Application Definition**:
  - **Connection Failure**: Verify host, port, bind DN, and SSL settings. Import FreeIPA CA cert to truststore if using LDAPS.
  - **Schema Errors**: Ensure `uid` is unique and `memberOf` is multi-valued. Use LDAP browser (e.g., Apache Directory Studio) to validate schema.
  - **Logs**: Check **Debug > Logs** or `/opt/tomcat/logs/iiq.log`.
- **Account Aggregation**:
  - **No Accounts Imported**: Confirm filter `(objectClass=posixAccount)` matches FreeIPA users. Test with `ldapsearch`.
  - **Correlation Issues**: Verify correlation rule in **Debug > Rules**. Check identity attribute mappings.
  - **Performance**: Limit `Max Records` for initial tests; increase for full sync.
- **General**: Use **Debug Pages** (`http://iiq.example.com:8080/identityiq/debug`) to inspect application and task objects.


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
- <img width="468" height="281" alt="image" src="https://github.com/user-attachments/assets/6e7084e7-1a32-4dd5-a060-0bdfc3b4da19" />


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


1. Test each feature with a small FreeIPA dataset (e.g., 10 users).
2. Expand configurations in `configs/freeipa.properties` for advanced settings.
3. Automate test data creation via `scripts/create_freeipa_users.sh`.
4. Schedule tasks for ongoing operations (aggregation, certifications, reports).
5. Explore advanced features (e.g., PAM, AI-driven analytics) if licensed.


