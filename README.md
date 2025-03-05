# Subject: Understanding Azure IAM Concepts

Hi [Recipient's Name],

I hope you are doing well. I wanted to share my understanding of Azure IAM concepts, covering key terms and their relationships.

### Key Concepts

#### Principals
Principals are security entities that can be authenticated and authorized to access resources. They are:
- **Users** – Individual accounts with access.
- **Groups** – Collections of users (similar to entitlements in our case).
- **Service Principals** – Identities used by applications, tools, and services to access Azure resources.

Service Principals are further classified into:
- **Managed Identities** – Managed by Azure for secure access.
- **Applications** – App registrations in Azure AD with assigned permissions.

#### Roles
- **Directory Roles** – Permissions for Azure resources.
- **App Roles** – Permissions for applications in Azure.
  - Note: There is no single API to retrieve all app roles at once. We need to fetch them for each principal separately.

#### Entitlement Grants and Hierarchy
Direct assignments (not transitive) include:
- **Group-to-Group (EH)**
- **Role-to-Group (EH)**
- **Role-to-User (EG)**
- **Group-to-User (EG)**

#### Policies and Scopes
We also have **Privileged Identity Management (PIM)**, for which we are still working on retrieving data.

Let me know if you have any questions or need further clarification.

Best regards,  
[Your Name]