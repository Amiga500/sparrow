# Security Recommendations for Sparrow Bitcoin Wallet

**Date:** 2026-02-10  
**Priority Order:** Critical → High → Medium → Low

---

## 🔴 CRITICAL PRIORITY (Fix Immediately)

### 1. Add TLS Hostname Verification

**Risk Level:** HIGH - Man-in-the-Middle Attack Vulnerability  
**File:** `src/main/java/com/sparrowwallet/sparrow/net/TcpOverTlsTransport.java`  
**Lines:** 74-86

#### Problem
The custom `X509TrustManager` does not verify that the certificate's hostname matches the server hostname on first connection. This allows an attacker to present a valid certificate for a different domain.

#### Current Code (Vulnerable)
```java
public void checkServerTrusted(X509Certificate[] certs, String authType) throws CertificateException {
    if(certs.length == 0) {
        throw new CertificateException("No server certificate provided");
    }
    try {
        certs[0].checkValidity();  // ❌ No hostname verification
    } catch(CertificateExpiredException e) {
        if(Storage.getCertificateFile(server.getHost()) == null) {
            throw new UnknownCertificateExpiredException(e.getMessage(), certs[0]);
        }
    }
}
```

#### Recommended Fix
```java
public void checkServerTrusted(X509Certificate[] certs, String authType) throws CertificateException {
    if(certs.length == 0) {
        throw new CertificateException("No server certificate provided");
    }
    
    // Add hostname verification
    try {
        // Verify the certificate is for the correct host
        String hostname = server.getHost();
        
        // Check CN (Common Name)
        String dn = certs[0].getSubjectX500Principal().getName();
        if(!verifyCertificateHost(dn, hostname)) {
            throw new CertificateException("Certificate hostname mismatch. Expected: " + hostname);
        }
        
        // Check SAN (Subject Alternative Names) if present
        Collection<List<?>> sans = certs[0].getSubjectAlternativeNames();
        if(sans != null && !sans.isEmpty()) {
            boolean sanMatch = false;
            for(List<?> san : sans) {
                if(san.size() >= 2 && san.get(0).equals(2)) { // Type 2 = DNS name
                    String dnsName = (String) san.get(1);
                    if(hostname.equalsIgnoreCase(dnsName) || 
                       wildcardMatch(hostname, dnsName)) {
                        sanMatch = true;
                        break;
                    }
                }
            }
            if(!sanMatch) {
                throw new CertificateException("Certificate SAN does not match hostname: " + hostname);
            }
        }
        
        certs[0].checkValidity();
    } catch(CertificateExpiredException e) {
        if(Storage.getCertificateFile(server.getHost()) == null) {
            throw new UnknownCertificateExpiredException(e.getMessage(), certs[0]);
        }
    } catch(CertificateParsingException e) {
        log.warn("Error parsing certificate", e);
        throw new CertificateException("Error parsing certificate", e);
    }
}

private boolean verifyCertificateHost(String dn, String hostname) {
    // Extract CN from DN
    String cn = null;
    for(String part : dn.split(",")) {
        String trimmed = part.trim();
        if(trimmed.startsWith("CN=")) {
            cn = trimmed.substring(3);
            break;
        }
    }
    return cn != null && (hostname.equalsIgnoreCase(cn) || wildcardMatch(hostname, cn));
}

private boolean wildcardMatch(String hostname, String pattern) {
    if(!pattern.startsWith("*.")) {
        return false;
    }
    String suffix = pattern.substring(2);
    return hostname.endsWith(suffix) && 
           hostname.indexOf('.') == hostname.length() - suffix.length() - 1;
}
```

#### Alternative: Use Java's Built-in Hostname Verifier
```java
public void checkServerTrusted(X509Certificate[] certs, String authType) throws CertificateException {
    if(certs.length == 0) {
        throw new CertificateException("No server certificate provided");
    }
    
    // Use Java's default hostname verifier
    HostnameVerifier hostnameVerifier = HttpsURLConnection.getDefaultHostnameVerifier();
    SSLSession mockSession = createMockSSLSession(certs);
    
    if(!hostnameVerifier.verify(server.getHost(), mockSession)) {
        throw new CertificateException("Hostname verification failed for: " + server.getHost());
    }
    
    try {
        certs[0].checkValidity();
    } catch(CertificateExpiredException e) {
        if(Storage.getCertificateFile(server.getHost()) == null) {
            throw new UnknownCertificateExpiredException(e.getMessage(), certs[0]);
        }
    }
}
```

#### Testing
Create a test to verify hostname validation:
```java
@Test
void testHostnameVerificationFailsForWrongCertificate() throws Exception {
    // Create a certificate for "wrong.example.com"
    X509Certificate wrongCert = createSelfSignedCert("wrong.example.com");
    
    // Try to connect to "server.example.com"
    TcpOverTlsTransport transport = new TcpOverTlsTransport(...);
    
    assertThrows(CertificateException.class, () -> {
        transport.getTrustManagers(null)[0].checkServerTrusted(
            new X509Certificate[]{wrongCert}, "RSA"
        );
    });
}
```

#### Impact if Not Fixed
- Attacker can intercept first connection to Electrum server
- Steal Bitcoin transaction details
- Manipulate transaction data
- Potentially steal funds through transaction manipulation

---

### 2. Implement Hardware Wallet Certificate Validation

**Risk Level:** HIGH - Hardware Wallet Impersonation  
**Files:**
- `src/main/java/com/sparrowwallet/sparrow/io/keycard/KeycardApi.java` (Line 61)
- `src/main/java/com/sparrowwallet/sparrow/io/satochip/SatoCardApi.java` (Line 52)

#### Problem
TODO comments indicate device certificate validation is not implemented. Hardware wallets should verify their authenticity to prevent malicious device impersonation.

#### Current Code
```java
// TODO check device certificate
```

#### Recommended Implementation

```java
public void verifyDeviceCertificate(CardChannel cardChannel) throws CardException, CertificateException {
    // Request device certificate
    ResponseAPDU response = cardChannel.transmit(new CommandAPDU(
        CLA_PROPRIETARY, INS_GET_CERTIFICATE, 0x00, 0x00
    ));
    
    if(response.getSW() != SW_OK) {
        throw new CardException("Failed to retrieve device certificate: " + 
                               Integer.toHexString(response.getSW()));
    }
    
    byte[] certData = response.getData();
    
    // Parse certificate
    CertificateFactory cf = CertificateFactory.getInstance("X.509");
    X509Certificate deviceCert = (X509Certificate) cf.generateCertificate(
        new ByteArrayInputStream(certData)
    );
    
    // Verify certificate chain
    X509Certificate manufacturerCA = loadManufacturerCA();
    
    try {
        deviceCert.verify(manufacturerCA.getPublicKey());
        deviceCert.checkValidity();
        
        // Verify device certificate attributes
        verifyDeviceCertificateAttributes(deviceCert);
        
        log.info("Hardware wallet certificate validated successfully");
    } catch(Exception e) {
        log.error("Hardware wallet certificate validation failed", e);
        throw new CertificateException("Invalid hardware wallet certificate", e);
    }
}

private void verifyDeviceCertificateAttributes(X509Certificate cert) throws CertificateException {
    // Verify manufacturer OID or specific attributes
    byte[] manufacturerOID = cert.getExtensionValue("1.2.3.4.5.6");  // Replace with actual OID
    if(manufacturerOID == null) {
        throw new CertificateException("Missing manufacturer OID in device certificate");
    }
    
    // Additional checks...
}

private X509Certificate loadManufacturerCA() throws CertificateException {
    try(InputStream is = getClass().getResourceAsStream("/certs/manufacturer-ca.crt")) {
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        return (X509Certificate) cf.generateCertificate(is);
    } catch(IOException e) {
        throw new CertificateException("Failed to load manufacturer CA certificate", e);
    }
}
```

#### User Communication
If certificate validation fails, inform the user:
```java
Alert alert = new Alert(Alert.AlertType.WARNING);
alert.setTitle("Hardware Wallet Authentication Failed");
alert.setHeaderText("Unable to verify hardware wallet authenticity");
alert.setContentText(
    "This hardware wallet's certificate could not be validated. " +
    "It may be counterfeit or damaged. Proceeding may put your funds at risk.\n\n" +
    "Do you want to continue anyway?"
);

Optional<ButtonType> result = alert.showAndWait();
if(result.isPresent() && result.get() == ButtonType.OK) {
    // User accepted risk
    continueWithoutValidation();
} else {
    // Abort connection
    throw new SecurityException("Hardware wallet authentication rejected by user");
}
```

---

## 🟡 HIGH PRIORITY (Fix Within 1-2 Weeks)

### 3. Replace String Password Handling with SecureString

**Risk Level:** MEDIUM - Memory Exposure  
**Files:** Multiple files in terminal UI and database persistence

#### Problem
Passwords stored as `String` objects cannot be securely cleared from memory because Strings are immutable in Java. They may remain in heap memory until garbage collected.

#### Affected Files
1. `src/main/java/com/sparrowwallet/sparrow/terminal/wallet/SettingsDialog.java`
2. `src/main/java/com/sparrowwallet/sparrow/terminal/wallet/LoadWallet.java`
3. `src/main/java/com/sparrowwallet/sparrow/terminal/wallet/NewWalletDialog.java`
4. `src/main/java/com/sparrowwallet/sparrow/terminal/wallet/WalletDialog.java`
5. `src/main/java/com/sparrowwallet/sparrow/io/db/DbPersistence.java`

#### Current Code Pattern
```java
String password = builder.build().showDialog(SparrowTerminal.get().getGui());
// Password remains in memory indefinitely
```

#### Recommended Fix

**Step 1:** Update method signatures to accept `SecureString`:
```java
// DbPersistence.java
private void update(Storage storage, Wallet wallet, SecureString password) throws StorageException {
    try {
        // Use password
        String passwordStr = password.asString();
        // ... database operations ...
    } finally {
        // SecureString will be cleared by caller
    }
}
```

**Step 2:** Update terminal dialogs:
```java
// LoadWallet.java (and similar files)
char[] passwordChars = builder.build().showDialog(SparrowTerminal.get().getGui()).toCharArray();
SecureString password = new SecureString(passwordChars);

try {
    // Use password
    storage.loadWallet(password);
} finally {
    password.clear();  // Zeroes out memory
    Arrays.fill(passwordChars, '\0');  // Clear char array too
}
```

**Step 3:** Update password dialog builders to return char[]:
```java
public char[] getPasswordFromUser(String prompt) {
    PasswordDialog dialog = new PasswordDialog(prompt);
    Optional<char[]> result = dialog.showAndWait();
    return result.orElse(new char[0]);
}
```

#### Implementation Checklist
- [ ] Update `DbPersistence` methods to use `SecureString`
- [ ] Update all terminal dialog password handling
- [ ] Update password prompts to use char[] or SecureString
- [ ] Add `finally` blocks to ensure passwords are cleared
- [ ] Add unit tests to verify password clearing
- [ ] Review all `String password` usages in codebase

#### Testing
```java
@Test
void testPasswordIsCleared() throws Exception {
    SecureString password = new SecureString("test-password");
    
    try {
        // Use password
        doSomethingWithPassword(password);
    } finally {
        password.clear();
    }
    
    // Verify password is cleared
    assertThrows(IllegalStateException.class, () -> {
        password.asString();  // Should throw after clear()
    });
}
```

---

### 4. Fix Process Resource Leak

**Risk Level:** MEDIUM - Resource Exhaustion  
**File:** `src/main/java/com/sparrowwallet/sparrow/AppController.java`  
**Lines:** 1070-1078

#### Problem
Spawned process is not stored or managed, so it cannot be terminated if needed. This can lead to orphaned processes.

#### Current Code
```java
final ProcessBuilder builder = new ProcessBuilder(cmd);
if(OsType.getCurrent() == OsType.UNIX) {
    Map<String, String> env = builder.environment();
    env.remove("LD_LIBRARY_PATH");
}
builder.start();  // ❌ Process not stored
quit(event);
```

#### Recommended Fix
```java
private volatile Process restartProcess = null;

// In restart method:
final ProcessBuilder builder = new ProcessBuilder(cmd);
if(OsType.getCurrent() == OsType.UNIX) {
    Map<String, String> env = builder.environment();
    env.remove("LD_LIBRARY_PATH");
}

// Store process reference
restartProcess = builder.start();

// Add shutdown hook for cleanup
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    if(restartProcess != null && restartProcess.isAlive()) {
        log.info("Terminating restart process on shutdown");
        restartProcess.destroy();
        try {
            if(!restartProcess.waitFor(5, TimeUnit.SECONDS)) {
                log.warn("Restart process did not terminate gracefully, forcing");
                restartProcess.destroyForcibly();
            }
        } catch(InterruptedException e) {
            Thread.currentThread().interrupt();
            restartProcess.destroyForcibly();
        }
    }
}));

quit(event);
```

#### Alternative: Use CompletableFuture
```java
CompletableFuture.runAsync(() -> {
    try {
        Process process = builder.start();
        int exitCode = process.waitFor();
        if(exitCode != 0) {
            log.error("Restart process failed with exit code: {}", exitCode);
        }
    } catch(Exception e) {
        log.error("Error in restart process", e);
    }
});
```

---

## 🟢 MEDIUM PRIORITY (Fix Within 1 Month)

### 5. Improve Exception Logging

**Risk Level:** LOW - Debugging Difficulty  
**Multiple Files**

#### Problem
Some exceptions are silently ignored or logged without context.

#### Examples

**Empty Catch Blocks:**
```java
// Storage.java:434
try {
    if(type == PersistenceType.JSON && type.getInstance().isEncrypted(walletFile)) {
        return true;
    }
} catch(IOException e) {
    //ignore  // ❌ Silent failure
}
```

**Generic Exception Catching:**
```java
// AppController.java:1079
} catch(Exception e) {  // ❌ Too broad
    log.error("Error restarting application", e);
}
```

#### Recommended Fixes

**For empty catch blocks:**
```java
} catch(IOException e) {
    log.debug("Could not check if file {} is encrypted: {}", 
              walletFile.getName(), e.getMessage());
    // Continue checking other types
}
```

**For generic exceptions:**
```java
} catch(IOException e) {
    log.error("Failed to start restart process: {}", cmd, e);
    showErrorDialog("Restart Failed", "Could not restart application", e);
} catch(SecurityException e) {
    log.error("Insufficient permissions to start process: {}", cmd, e);
    showErrorDialog("Permission Denied", "Cannot restart application", e);
}
```

#### Systematic Fix Process
1. Search for all `catch` blocks with comments like `//ignore`
2. Add at minimum DEBUG level logging
3. Consider if exception should propagate
4. Add context to error messages

---

### 6. Add Input Validation for File Operations

**Risk Level:** MEDIUM - Path Traversal  
**Files:** Storage-related classes

#### Problem
File operations should validate inputs to prevent path traversal attacks.

#### Recommended Implementation

```java
public class SecureFileValidator {
    private static final Pattern SAFE_FILENAME = Pattern.compile("^[a-zA-Z0-9._-]+$");
    
    public static File validateWalletFile(File baseDir, String filename) throws SecurityException {
        // Validate filename
        if(!SAFE_FILENAME.matcher(filename).matches()) {
            throw new SecurityException("Invalid filename: " + filename);
        }
        
        // Ensure file is within baseDir
        File file = new File(baseDir, filename);
        try {
            String canonicalBase = baseDir.getCanonicalPath();
            String canonicalFile = file.getCanonicalPath();
            
            if(!canonicalFile.startsWith(canonicalBase)) {
                throw new SecurityException("Path traversal attempt detected: " + filename);
            }
        } catch(IOException e) {
            throw new SecurityException("Could not validate file path", e);
        }
        
        return file;
    }
}
```

**Usage:**
```java
public static File getWalletFile(String walletName) {
    return SecureFileValidator.validateWalletFile(getWalletsDir(), walletName + ".mv.db");
}
```

---

## 🔵 LOW PRIORITY (Ongoing Improvement)

### 7. Increase Test Coverage

**Current Coverage:** Limited (13 test files for 81k+ LOC)

#### Recommendations

**Phase 1: Security Tests**
```java
// SecurityTest.java
@Test
void testWalletEncryptionWithStrongPassword() { }

@Test
void testPasswordRequirements() { }

@Test
void testPrivateKeyClearing() { }

@Test  
void testCertificatePinning() { }
```

**Phase 2: Core Business Logic**
```java
// WalletTest.java
@Test
void testTransactionSigning() { }

@Test
void testFeeCalculation() { }

@Test
void testAddressGeneration() { }
```

**Phase 3: Integration Tests**
```java
// ElectrumServerTest.java
@Test
void testServerConnection() { }

@Test
void testServerFailover() { }
```

---

### 8. Add Security Documentation

Create the following files:

#### SECURITY.md
```markdown
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 2.4.x   | :white_check_mark: |
| 2.3.x   | :white_check_mark: |
| < 2.3   | :x:                |

## Reporting a Vulnerability

**Please DO NOT report security vulnerabilities through public GitHub issues.**

Email: security@sparrowwallet.com

You should receive a response within 48 hours. 

## Security Practices

- All wallet files are encrypted using ECIES
- Private keys are never stored in plain text
- Passwords use Argon2 key derivation
- Database encryption with AES
- TLS for network communication
- Certificate pinning for Electrum servers

## Known Limitations

- First connection to new Electrum server accepts any certificate (user must approve)
- Terminal UI uses String for passwords (scheduled for fix in v2.5.0)
```

---

## Implementation Priority Matrix

| Issue | Risk | Effort | Priority | Timeline |
|-------|------|--------|----------|----------|
| TLS Hostname Verification | HIGH | LOW | 🔴 Critical | Immediate |
| HW Certificate Validation | HIGH | MEDIUM | 🔴 Critical | 1 week |
| SecureString Migration | MEDIUM | HIGH | 🟡 High | 2 weeks |
| Process Management | MEDIUM | LOW | 🟡 High | 1 week |
| Exception Logging | LOW | LOW | 🟢 Medium | 1 month |
| Input Validation | MEDIUM | MEDIUM | 🟢 Medium | 1 month |
| Test Coverage | LOW | HIGH | 🔵 Low | Ongoing |
| Security Docs | LOW | LOW | 🔵 Low | 1 month |

---

## Review Checklist for Implementation

After implementing each fix, verify:

- [ ] Code review completed
- [ ] Unit tests added
- [ ] Integration tests pass
- [ ] Manual testing completed
- [ ] Documentation updated
- [ ] Security team review (if applicable)
- [ ] Merged to main branch

---

**Document Version:** 1.0  
**Last Updated:** 2026-02-10  
**Next Review:** After critical issues addressed
