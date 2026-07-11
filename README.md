# Configuration Blueprint: Passwordless Workload Identity & Secret Extraction

## Phase 1: Establishing the Control Rooms & Access Boundaries

### The Plain-Talk Reality
Before touching a single script, you have to separate your management tools. You are dealing with two entirely different administrative spaces: the "ID office" where digital entities get identity cards and background checks, and the "factory floor" where real infrastructure assets exist. Blurring these two together is why most deployments fail; you must know exactly which console governs which object.

### The Concrete Configuration Steps

#### 1. The Microsoft Entra Admin Center (The Identity Control Room)
*   **What you are doing:** Accessing the core directory service to manage security principals.
*   **The Object:** The Microsoft Entra ID Tenant.
*   **Why:** This serves as the root authority. Every app registration, permission scope, and security token we issue is bound to this specific Directory (tenant) ID.

#### 2. The Azure Portal (The Infrastructure Control Room)
*   **What you are doing:** Navigating to the resource management plane (`portal.azure.com`).
*   **The Object:** The Azure Resource Group (`RG-AUTOMATION-MESH-PROD`) and Azure Subscription.
*   **Why:** This acts as the physical container for your assets. The Subscription ID acts as your broad infrastructure boundary, while the Resource Group functions as the logical boundary where security policies and Role-Based Access Control (RBAC) are enforced.

  ### The Universal Blueprint: Decoupling Identity from Infrastructure

While this specific configuration uses the Azure Portal as the infrastructure destination, this model defines a universal, cross-cloud architecture for machine-to-machine (M2M) and autonomous AI security. Microsoft Entra ID acts as your immutable, global Identity Control Center—the absolute root of trust. The infrastructure control room, however, is completely modular. In a production AI ecosystem, the target "factory floor" could just as easily be an AWS environment, a Google Cloud platform, a Kubernetes cluster, or a third-party vector database hosting proprietary LLM training data. Regardless of the destination vendor, the security topology remains identical: Microsoft Entra ID handles the heavy lifting of federating trust and verifying the inbound identity claims of the automated agent, while the target platform merely enforces its local access boundaries. Mastering this relationship means you can secure any non-human workload or autonomous AI agent across the entire modern enterprise footprint, using the exact same flow of events.  

---

## Phase 2: Generating Identity Credentials & Digital Coordinates

### The Plain-Talk Reality
An automation engine like GitHub Actions is a stranger to Azure. To allow it in, you must create a digital representation of that runner inside your identity directory—essentially creating a "bot account." Once created, you must collect the exact matching coordinate strings so that your automation script knows precisely who it is claiming to be and where it needs to knock.

### The Concrete Configuration Steps

#### 1. Register the Application Object
*   **Path:** Entra Admin Center -> Identity -> Applications -> App registrations -> New registration.
*   **Action:** Name the object `app-automation-bot` and select *Accounts in this organizational directory only*.
*   **The Object:** Application Object & Service Principal Object.
*   **Why:** The Application Object acts as the global blueprint for your bot, while Entra ID automatically generates a corresponding Service Principal object in your tenant to act as the local "user account" that can be granted actual permissions.
*   **Coordinate Captured:** **Application (client) ID**.

#### 2. Extract the Directory & Subscription Coordinates
*   **Path A:** Entra Admin Center -> Identity -> Overview. Copy the **Directory (tenant) ID**.
*   **Path B:** Azure Portal -> Resource Groups -> `RG-AUTOMATION-MESH-PROD` -> Overview. Copy the **Subscription ID**.
*   **Why:** Your GitHub script cannot route an authentication request into a vacuum; it requires the Tenant ID to find your specific directory, and the Subscription ID to locate your specific infrastructure cluster.

---

## Phase 3: Building the Passwordless Trust Link (Federated OIDC)

### The Plain-Talk Reality
Traditional security relies on copy-pasting a password (client secret) from Azure into GitHub. If that password leaks, your entire environment is compromised. Instead, we use Workload Identity Federation (WIF). We tell Azure to trust GitHub's internal token office. You establish a strict rule: *"If a request arrives from a specific organization, a specific repository name, and a specific branch, trust it automatically without a password."*

### The Concrete Configuration Steps

#### 1. Configure the Federated Identity Credential
*   **Path:** Entra Admin Center -> App registrations -> `app-automation-bot` -> Certificates & secrets -> Federated credentials -> + Add credential.
*   **Scenario Selection:** Select **GitHub Actions deploying Azure resources** from the dropdown menu.
*   **The Configuration Settings:**
    *   **Organization:** `Compcode1` *(Must match your exact GitHub username case-sensitively)*.
    *   **Repository:** `key-vault-zero-trust` *(The target code repository)*.
    *   **Entity type:** Set to **Branch**.
    *   **Branch name:** `main`.
*   **The Object:** Federated Identity Credential Policy.
*   **Why:** This writes an explicit verification rule into Entra ID. When GitHub passes its OpenID Connect (OIDC) token assertion, Entra ID decrypts the token, reads the Subject Claim (`repo:Compcode1/key-vault-zero-trust:ref:refs/heads/main`), and performs a strict binary string match. If a single character or casing is mismatched, access is instantly denied.

---

## Phase 4: Granting Resource Access & Pipeline Execution

### The Plain-Talk Reality
An identity card is useless if you aren't allowed past the front door of the building. Once the bot has an identity and a passwordless handshake, you must explicitly grant that bot permission to look inside your secret vault. Finally, you write the automation script that tells GitHub how to present its identity card, log in, and securely pull the data without leaking it to the public.

### The Concrete Configuration Steps

#### 1. Assign Data-Plane Permissions on Key Vault
*   **Path:** Azure Portal -> Key Vaults -> `key-vault-sgt` -> Access control (IAM) or Access Policies.
*   **Action:** Assign the **Key Vault Secrets User** role to the `app-automation-bot` service principal.
*   **The Object:** Role-Based Access Control (RBAC) Assignment.
*   **Why:** Authenticating to Azure only gets the bot through the perimeter. To execute a data-plane transaction like reading a secret value (`SecretGet`), the bot's identity object must be explicitly bound to a data-plane role containing the `Microsoft.KeyVault/vaults/secrets/getSecret/action` permission.

#### 2. Deploy and Execute the Automation Pipeline
*   **Path:** GitHub Repository -> `.github/workflows/extract-secret.yml`.
*   **Action:** Write the YAML workflow utilizing the `azure/login@v2` action block populated with your three collected ID variables, followed by `azure/get-keyvault-secrets@v1`.
*   **The Execution Flow:**
    1.  The workflow runner requests a cryptographically signed OIDC token from GitHub's token authority.
    2.  The runner executes `azure/login@v2`, presenting that OIDC token to Microsoft Entra ID.
    3.  Entra ID validates the federated trust string, matches it, and returns a short-lived Azure Access Token (AT).
    4.  The runner uses that Access Token to hit the Key Vault data plane, extracts the secret (`secret-sgt`), and automatically masks the output string in the console logs (`***`) to protect the raw payload from visibility.
