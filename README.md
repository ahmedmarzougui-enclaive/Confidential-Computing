# Confidential Computing + AMD SEV-SNP + vHSM (Summary)

Confidential Computing keeps data safe while it’s being used, not just at rest or in transit. It runs sensitive code inside a Trusted Execution Environment (TEE) or enclave.

## TEE in Action

- Memory Encryption: RAM inside the enclave is encrypted.
- Attestation: Proves the enclave is authentic and running trusted code.
- Confidential Boot: Only verified components load before the enclave starts.
- Sealing/Binding: Data is encrypted so only that enclave/device can use it.
- Secret Provisioning: Keys are delivered directly into the enclave without exposure.

“Encrypted RAM + Proof + Safe Start + Locked Data + Hidden Keys” = TEE in action.

## AMD SEV-SNP (Secure Encrypted Virtualization – Secure Nested Paging)

- Memory encryption: All VM memory is encrypted by CPU.
- Integrity protection: Stops memory rollback or tampering.
- Secure nested paging: Hypervisor cannot map or access VM memory.
- Attestation: Remote verification that VM runs inside a protected enclave.

### How it works

- VM memory & CPU registers are encrypted automatically.
- CPU enforces access rules via RMP & page validation.
- Hypervisor or host cannot access VM memory (even VMPL0).
- Remote attestation proves VM runs securely inside SEV-SNP.

## vHSM’s Role

- Acts as a secure key vault inside the VM.
- Keys are generated and stay inside the vHSM; they never leave protected memory.
- SEV-SNP ensures vHSM memory is encrypted and tamper-proof.
- Applications request keys from vHSM; hypervisor or cloud provider cannot read them.

## Secret Request & Provisioning Flow (with Nitride verifying attestation)

```mermaid
sequenceDiagram
    autonumber
    participant App as Application (inside VM)
    participant Nitride as Nitride (vHSM + Agent)
    participant VM as SEV-SNP VM
    participant Attest as Attestation Service (Verifier)
    participant Prov as Provisioning Service / KMS

    Note over VM: SEV-SNP active; Confidential Boot; Encrypted RAM; Integrity protected

    App->>Nitride: Request secret (key/certificate)

    Note over Nitride: Do not release secret blindly; require attestation proof

    Nitride->>Attest: Request attestation (SEV-SNP report with nonce)
    Attest-->>Nitride: Signed attestation result (VM ID, CPU/SEV-SNP version, code hash, timestamp)

    Note over Nitride: Verify attestation locally (policy checks: OVMF/firmware, TCB, measurements)

    alt Attestation verified
        Nitride->>Prov: Present approval; request secret
        Prov->>Nitride: Deliver secret directly into protected memory
        Nitride-->>App: Secret usable (perform crypto); secret never leaves Nitride/vHSM
    else Attestation failed
        Nitride-->>App: Deny request (secret unavailable)
    end

    Note over VM,Nitride: Hypervisor/host cannot read VM or Nitride memory (RMP, VMPL)
```

### Explanation aligned to the flow you provided
- VM wants a secret: The application asks Nitride (which includes vHSM functionality) for a key/certificate.
- Nitride verifies the VM: It requires a valid SEV-SNP attestation before releasing or generating secrets.
- Attestation report request: Nitride requests a report and sends it to the Attestation Service.
- Attestation Service response: Returns a signed result confirming the VM is trusted (VM ID, CPU & SEV-SNP version, hash of running code, timestamp).
- Secret provisioning: After Nitride verifies the attestation, the Provisioning Service delivers the requested secret directly into Nitride/vHSM.
- VM gets the secret: The application can use the secret via Nitride/vHSM; the secret never leaves protected memory.
