# AWS Cloud GRC: Three-Layered Identity & Network Perimeter Defense for Sensitive Data

## 📌 Project Overview

This project implements a Cloud Governance, Risk, and Compliance (GRC) security framework around a sensitive AWS S3 data asset (`grc-secure-financial-vault-2026`), modeled as a fictional financial records store for portfolio purposes.

In an enterprise environment, high-level corporate risk metrics must translate into enforceable cloud security controls. This project applies the **Principle of Least Privilege (PoLP)** and zero-trust network architecture concepts to defend sensitive data against credential theft and unauthorized manipulation.

> **Note on scope:** this is a self-directed lab environment built to demonstrate GRC control design and validation methodology, not a production deployment. IP ranges, account IDs, and log excerpts below use placeholder/example values (e.g. the TEST-NET-1 documentation range `192.0.2.0/24`) and are marked accordingly.

## 📂 Repository Structure

```
├── policies/
│   ├── s3-perimeter-bucket-policy.json          # Layer 3: Network Boundary (bucket policy)
│   ├── financial-admin-least-privilege-policy.json  # Layer 2: Scoped admin permissions
│   └── financial-admin-trust-policy.json        # IAM Role Trust Relationship Definition
├── img/
│   ├── auditor-delete-denied.png                # Visual Proof: Integrity Verification
│   └── external-ip-admin-denied.png              # Visual Proof: Perimeter Verification
├── main.tf                                       # Infrastructure-as-Code (Compliance as Code)
└── README.md                                     # GRC Control Framework & Audit Trail Logs
```

## 🛡️ Control Architecture (The 3 Layers)

1. **Layer 1: The Auditor Role (Read-Only Control)** – Grants access to inspect compliance without risking data integrity (Confidentiality).
2. **Layer 2: The Admin Role (Scoped Management Control)** – Grants engineers only the specific S3 actions needed to manage lifecycle rules, versioning, and encryption settings — not blanket `s3:*` access (Availability, Least Privilege).
3. **Layer 3: Explicit Deny Policy (Network Perimeter Control)** – A global safeguard overriding all permissions. Even if administrative credentials are stolen, data access is blocked unless the request originates from an approved network CIDR range.

---

## 🗺️ GRC Framework Mapping

| Control ID    | Risk Metric                         | Technical Control Implemented                                     | CIA Triad         | Impact        |
| ------------- | ------------------------------------ | ------------------------------------------------------------------ | ------------------ | ------------- |
| **FIN-S3-01** | Unauthorized Data Exfiltration       | `Financial-Auditor-Role` with `AmazonS3ReadOnlyAccess`             | Confidentiality    | **Mitigated** |
| **FIN-S3-02** | Accidental/Malicious Deletion        | Scoped admin policy + MFA-conditioned delete + IAM boundary block  | Integrity          | **Mitigated** |
| **FIN-S3-03** | Credential Theft / Perimeter Breach  | S3 Bucket Policy with `Explicit Deny` on `NotIpAddress`             | Perimeter Defense  | **Mitigated** |
| **FIN-S3-04** | Excess Standing Privilege            | Least-privilege scoped policy replacing `AmazonS3FullAccess`        | Confidentiality/Integrity | **Mitigated** |
| **FIN-S3-05** | Data-at-Rest Exposure                | Default encryption (SSE-KMS) + versioning enabled on the bucket    | Confidentiality    | **Mitigated** |

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: Establish the Target Asset

1. Navigated to the **Amazon S3 Console** and created a uniquely named bucket: `grc-secure-financial-vault-2026`.
2. Enforced strict compliance by keeping **Block Public Access** fully enabled to eliminate accidental exposure.
3. Enabled **default encryption (SSE-KMS)** and **versioning** to protect data at rest and support recovery from accidental overwrite/delete.

### Step 2: Configure Layer 1 (The Auditor Role)

1. Created an IAM Role named `Financial-Auditor-Role` using the local AWS account as the trusted entity.
2. Attached the AWS-managed policy `AmazonS3ReadOnlyAccess`.
3. *Result:* Users assuming this role can read logs and verify configurations but cannot delete or overwrite evidence.

### Step 3: Configure Layer 2 (The Admin Role — Least Privilege)

1. Created an IAM Role named `Financial-Admin-Role`, trusted only by a defined principal and requiring MFA to assume (see `financial-admin-trust-policy.json`).
2. Attached a **custom scoped policy** (`financial-admin-least-privilege-policy.json`) instead of the AWS-managed `AmazonS3FullAccess` policy. The custom policy:
   - Allows object read/write/versioned-get and lifecycle/versioning/encryption configuration — the actions actually required to administer the bucket
   - Explicitly denies changes to the bucket policy and ACLs, so an admin identity cannot silently weaken Layer 3's perimeter control
   - Requires MFA to be present for delete actions, adding a control against a single stolen credential being enough to destroy data
3. *Result:* the admin role can do its job without holding blanket `s3:*` — a direct fix for the standing-privilege risk that a fully-managed policy introduces.

### Step 4: Enforce Layer 3 (The Hard Network Boundary)

1. Navigated to the S3 bucket permissions tab and deployed the following **Bucket Policy** (`s3-perimeter-bucket-policy.json`).
2. This policy uses a `Deny` effect tied to a `NotIpAddress` condition block to isolate data access to an approved CIDR range.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceCorporateNetworkBoundary",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::grc-secure-financial-vault-2026",
        "arn:aws:s3:::grc-secure-financial-vault-2026/*"
      ],
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": ["192.0.2.0/24"]
        }
      }
    }
  ]
}
```
*(`192.0.2.0/24` is the reserved TEST-NET-1 documentation range, used here as a placeholder for a real corporate CIDR block.)*

---

## 🧪 Control Validation & Empirical Evidence

To verify operational effectiveness, compliance tests were executed across all access vectors.

### Validation Test Log

| Test ID    | Assumed Identity         | Source IP Address     | Attempted Action  | Expected GRC Result | Actual Result      | Status   |
| ---------- | ------------------------- | ---------------------- | ------------------ | --------------------- | -------------------- | -------- |
| **TST-01** | `Financial-Auditor-Role`  | Approved Corp IP       | Read Object         | **Allowed**            | **Allowed**           | **PASS** |
| **TST-02** | `Financial-Auditor-Role`  | Approved Corp IP       | Delete Object        | **Denied**             | **Access Denied**     | **PASS** |
| **TST-03** | `Financial-Admin-Role`    | Approved Corp IP       | Put/Write Object     | **Allowed**            | **Allowed**           | **PASS** |
| **TST-04** | `Financial-Admin-Role`    | Untrusted External IP  | List/Read Bucket     | **Denied**             | **Access Denied**     | **PASS** |
| **TST-05** | `Financial-Admin-Role`    | Approved Corp IP, no MFA | Delete Object       | **Denied**             | **Access Denied**     | **PASS** |
| **TST-06** | `Financial-Admin-Role`    | Approved Corp IP       | Modify Bucket Policy | **Denied**             | **Access Denied**     | **PASS** |

### Simulated Audit Trail (illustrative CloudTrail-format event)

The excerpt below is a hand-constructed example in CloudTrail's event format, used to illustrate what a real export would show for **TST-04** — it is not a live export from an AWS account.

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AssumedRole",
    "arn": "arn:aws:iam::123456789012:assumed-role/Financial-Admin-Role/Auditor-Test",
    "sessionContext": {
      "sessionIssuer": { "userName": "Financial-Admin-Role" }
    }
  },
  "eventTime": "2026-08-10T14:30:00Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "ListObjects",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.12",
  "errorCode": "AccessDenied",
  "errorMessage": "Access Denied"
}
```

## 🚀 Key Takeaways & Competencies Demonstrated

- **Zero-Trust Network Architecture**: Treating credentials as insufficient for authorization without matching network conditions.
- **AWS IAM**: Context-aware policy evaluation, scoped custom policies, trust policies with MFA conditions, and explicit deny logic.
- **Least Privilege in Practice**: Identifying and correcting an over-permissioned managed policy with a custom scoped alternative — and documenting why.
- **Audit Readiness**: Structured control mapping and validation logs matching SOC 2 / ISO 27001 evidence request formats.
