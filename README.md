# SPIFFE-FPGA Formal Verification

This repository contains a formal security verification of a Zero Trust protocol SPIFFE-FPGA for multi-tenant FPGA cloud using ProVerif. 
The protocol enables secure deployment and attestation of workloads on FPGAs with cryptographic identity management.

## Protocol Overview

The protocol involves several key components working together to establish a secure deployment and runtime environment:

### Components

1. **Tenant**: The entity that owns and deploys FPGA designs
    - Sends designs to TEE for validation
    - Authenticates with FPGA for deployment
    - Establishes secure channels for bitstream transfer

2. **TEE (Trusted Execution Environment)**: Validates designs before deployment
    - Provides attestation of its secure state
    - Validates and signs design hashes
    - Issues validation certificates

3. **FPGA/Security Agent**: The target deployment platform
    - Performs node attestation with AIK (Attestation Identity Key)
    - Manages isolated regions for workload execution
    - Handles runtime attestation of workloads

4. **SPIFFE Server**: Identity management authority
    - Issues SVIDs (SPIFFE Verifiable Identity Documents)
    - Verifies node and workload attestations
    - Maintains trust bundles for the infrastructure

5. **Workloads**: User applications running in isolated FPGA regions
    - Perform attestation to receive SVIDs
    - Establish mTLS connections with other workloads
    - Run in cryptographically isolated regions

### Protocol Flow

1. **TEE Attestation**: TEE provides attestation to Tenant
2. **Design Validation**: Tenant sends encrypted design to TEE for validation
3. **Deployment Request**: Tenant requests FPGA allocation with validated design
4. **Node Attestation**: FPGA performs node attestation to SPIFFE server and agent
5. **Mutual Authentication**: Tenant and FPGA authenticate each other
6. **Secure Deployment**: Encrypted bitstream transferred over authenticated channel
7. **Workload Attestation**: Deployed workloads attest to receive SVIDs
8. **mTLS Communication**: Workloads establish secure channels using SVIDs

## Formal Verification Results

The ProVerif verification confirms the following security properties:

### ✅ Design Confidentiality
```
Query not attacker(secret_design[]) is true.
```
The secret FPGA design remains confidential throughout the protocol execution. Attackers cannot obtain the design even with full network access.

### ✅ TEE Validation Chain Integrity
```
Query event(TEEValidatesDesign(d,h)) ==> event(TenantSendsDesign(d,n,sid)) is true.
Query event(TEEValidatesDesign(d,h)) ==> event(TEEReceivesDesign(d,n,sid)) is true.
Query event(TEEReceivesDesign(d,n,sid)) ==> event(TenantSendsDesign(d,n,sid)) is true.
```
The TEE validation process maintains proper event ordering. Designs are only validated after being properly sent and received through authenticated channels.

### ✅ TEE Attestation Authentication
```
Query event(TenantVerifiesTEE(tee_pk,tee_nonce,tee_attestation)) ==> 
      event(TEEProvidesAttestation(tee_pk,tee_nonce,tee_attestation)) is true.
```
TEE attestations verified by the Tenant are authentic and cannot be forged by attackers.

### ✅ Bitstream Integrity
```
Query event(FPGAVerifiesBitstream(d,h)) ==> event(TEEValidatesDesign(d,h)) is true.
```
FPGAs only accept bitstreams that have been validated by the TEE, ensuring design integrity.

### ✅ Node Attestation Security
```
Query event(NodeAttestationVerified(fpga_serial,aik_pub)) ==> 
      event(NodeAttestationStarted(fpga_serial,aik_pub,n)) is true.
```
Node attestations can only be verified for FPGAs that actually initiated the attestation process. This prevents attestation hijacking attacks.

### ✅ Serial Number Binding
```
Query not (event(NodeAttestationStarted(fpga_serial1,aik_pub1,n1)) && 
           event(NodeAttestationVerified(fpga_serial2,aik_pub2)) && 
           fpga_serial1 ≠ fpga_serial2) is true.
```
FPGA serial numbers are properly bound to attestations, preventing cross-device attestation attacks.

### ✅ Workload Attestation Chain
```
Query event(WorkloadSVIDIssued(id,rid,k)) ==> 
      event(WorkloadAttestationVerified(h,rid,k)) is true.
```
Workload SVIDs are only issued after successful attestation verification.

### ✅ Key Lifecycle Tracking
```
Query event(KeyCompromised(rid,k)) ==> event(RegionKeyGenerated(rid,k)) is true.
```
Any compromised keys can be traced back to their generation event, enabling forensic analysis.

### ✅ mTLS Authentication
```
Query event(mTLSEstablished(id1,id2)) ==> 
      event(WorkloadSVIDIssued(id1,rid1,k1)) && 
      event(WorkloadSVIDIssued(id2,rid2,k2)) is true.
```
mTLS connections require both parties to have valid SVIDs issued by the SPIFFE Server.

### ✅ SVID Uniqueness
```
Query event(WorkloadSVIDIssued(id,rid1,k1)) && 
      event(WorkloadSVIDIssued(id,rid2,k2)) ==> 
      rid1 = rid2 && k1 = k2 is true.
```
Each SPIFFE ID maps to a unique region and key pair, preventing identity confusion.

### ✅ Runtime Measurement Integrity
```
Query event(RuntimeMeasurement(rid,h1)) && 
      event(RuntimeMeasurement(rid,h2)) ==> 
      h1 = h2 is true.
```
Runtime measurements for a given region remain consistent, detecting any tampering attempts.

## Security Guarantees

The formal verification provides strong security guarantees:

1. **End-to-End Design Protection**: Designs remain encrypted from Tenant to FPGA
2. **Attestation Chain Integrity**: Each step in the attestation chain is cryptographically verified
3. **Identity Uniqueness**: SPIFFE IDs uniquely identify workloads with no ambiguity
4. **Isolation**: Workloads run in cryptographically isolated regions
5. **Tamper Detection**: Any modification to runtime state is detectable
6. **Authentication**: All parties are mutually authenticated before sensitive operations

## Running the Verification

To verify the protocol:

```bash
proverif spiffe-fpga.pv
```

The verification should complete with all security properties confirmed as `true`.

## Citation

If you use this protocol or verification results in your research, please cite:
```bibtex
@inproceedings{FPT-2025,
  title={SPIFFE-FPGA: A Zero Trust Architecture for Multi-Tenant FPGAs using SPIFFE},
  author={[Authors]},
  year={2025}
}
```