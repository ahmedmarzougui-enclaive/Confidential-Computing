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

## Attestation & Local Key Generation Flow (with Nitride)

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant vHSM as vHSM (in SEV-SNP VM)
    participant VM as SEV-SNP VM
    participant Nitride as Nitride (VM Agent)
    participant Attest as Attestation Service (Verifier)

    Note over VM: Confidential boot OK. SEV-SNP active. Encrypted RAM. Integrity protected.

    App->>vHSM: Request key or crypto operation

    alt Key already exists
        vHSM-->>App: Perform crypto (keys never leave vHSM)
    else Key absent (policy requires attestation before generation/use)
        vHSM-->>Nitride: Gate key generation/use pending attestation
        Nitride->>Attest: Submit SEV-SNP attestation report (with nonce)
        Attest-->>Nitride: Approval token (measurements verified)
        Nitride-->>vHSM: Attestation approved (enable key generation/use)
        vHSM->>vHSM: Generate key X locally (sealed to VM policy/TCB)
        vHSM-->>App: Perform crypto using key X
    end

    Note over VM,vHSM: Hypervisor cannot read VM memory (RMP, VMPL). Keys remain inside vHSM.
```

### Where Nitride fits (local-key scenario)
- Runs inside the VM as the attestation/provisioning gatekeeper.
- Verifies the VM via SEV-SNP attestation before vHSM is allowed to generate or use certain keys.
- No external key provisioning needed; vHSM generates and stores keys locally after Nitride signals attestation approval.
