# Confidential Computing + AMD SEV-SNP + vHSM (Summary)

Confidential Computing keeps data safe while it’s being used, not just at rest or in transit. It runs sensitive code inside a Trusted Execution Environment (TEE) or enclave.

## TEE in Action

- Memory Encryption: RAM inside the enclave is encrypted.
- Attestation: Proves the enclave is authentic and running trusted code.
- Confidential Boot: Only verified components load before the enclave starts.
- Sealing/Binding: Data is encrypted so only that enclave/device can use it.
- Secret Provisioning: Keys are delivered directly into the enclave without exposure.

“Encrypted RAM + Proof + Safe Start + Locked Data + Hidden Keys” = TEE in action.

## AMD SEV-SNP (Secure Encrypted Virtualization - Secure Nested Paging)

- Memory encryption: All VM memory is encrypted by CPU.
- Integrity protection: Stops memory rollback or tampering.
- Secure nested paging: Hypervisor cannot map or access VM memory.
- Attestation: Remote verification that VM runs inside a protected enclave.

# How AMD SEV-SNP Works

## 1. VM Memory and CPU Registers Are Encrypted Automatically
- SEV-SNP encrypts all VM RAM using a hardware key stored inside the AMD CPU.
- Even if the hypervisor tries to read the VM’s RAM, it only sees encrypted data.
- CPU registers are also encrypted whenever the VM is paused or switched, so the host cannot see them.

## 2. CPU Enforces Access Rules via RMP (Reverse Map Table) and Page Validation
- The CPU uses a hardware table called the Reverse Map Table (RMP) to decide which VM or process can access each memory page.
- Every memory access must pass an RMP check.
- If the page does not belong to your VM or is not allowed for your privilege level, the CPU blocks the access instantly.

## 3. Hypervisor or Host Cannot Access VM Memory (Even at VMPL0)
- SEV-SNP introduces VMPL (Virtual Machine Privilege Levels).
- Even the hypervisor, which normally runs at the highest privilege, is not allowed to read or modify SNP-protected memory.
- The CPU enforces this restriction in hardware, so the cloud provider cannot bypass it.

## 4. Remote Attestation Proves the VM Runs Securely Inside SEV-SNP
- The VM can generate a signed attestation report from the AMD Security Processor.
- This report contains measurements of the VM’s initial state (bootloader, kernel, configs).
- External services (like Nitride, KMS, or a provisioning service) verify the report before sending secrets.
- This ensures only a genuine, untampered Confidential VM receives sensitive data.

## vHSM’s Role

- Acts as a secure key vault inside the VM.
- Keys are generated and stay inside the vHSM; they never leave protected memory.
- SEV-SNP ensures vHSM memory is encrypted and tamper-proof.
- Applications request keys from vHSM; hypervisor or cloud provider cannot read them.

## When do you need attestation?

- Not needed for intra-VM calls: If the application is inside the same VM and the key is already present in vHSM (and your policy allows), the app can use it directly. No attestation step is required for that local use.
- Needed for external secrets or gated policies: If a key must be released by an external KMS/provisioning service, or your policy requires proving VM trust before generating/using the key, Nitride performs attestation first.

## Case A: Key already in vHSM (no attestation needed)

```mermaid
sequenceDiagram
    participant App as Application
    participant vHSM as vHSM
    App->>vHSM: Use key X (encrypt/sign)
    vHSM-->>App: Result (key stays inside vHSM)
```

## Case B: Key missing or policy requires attestation (Nitride verifies)

```mermaid
sequenceDiagram
    participant App as Application
    participant vHSM as vHSM
    participant Nitride as Nitride Agent
    participant Attest as Attestation Service
    participant Prov as Provisioning Service
    App->>vHSM: Use key X
    vHSM-->>Nitride: Key X missing or gated by policy
    Nitride->>Attest: Request SNP attestation (with nonce)
    Attest-->>Nitride: Approval (verified measurements)
    Nitride->>Prov: Request secret X with approval
    Prov->>vHSM: Inject secret X into protected memory
    vHSM-->>App: Result (key never leaves vHSM)
```

Notes:
- In Case A, only local access controls apply (e.g., socket ACLs, mTLS, process/cgroup identity). No remote attestation needed.
- In Case B, Nitride handles attestation and secret provisioning. The app still never sees raw keys; they remain in vHSM memory.


Nitride – Role 
Nitride is a VM-resident agent that acts as the bridge between vHSM and external secret sources.

Its main functions:
Attestation: Proves to the provisioning service that the VM is running securely inside a SEV-SNP enclave.
Secret Request: Once attestation succeeds, Nitride requests the needed secret/key (e.g., LUKS passphrase, API key) from the external provisioning service.
Secure Injection: Delivers the secret directly into vHSM’s protected memory—keys never leave the enclave.

 “Nitride = the secure messenger that proves trust and fetches secrets.”

In Case A: Nitride does nothing because the key already exists.
In Case B: Nitride ensures that only a verified, trusted VM receives secrets.
