# Security Implementation Guide - Life OS Integrations

## Overview

This document provides comprehensive security best practices, OAuth 2.0 implementation patterns, token management strategies, secure storage implementations, and security audit checklists for all Life OS integrations.

**Last Updated**: December 2024  
**Security Level**: Production-ready

---

## OAuth 2.0 Security Best Practices

### Authorization Code Flow with PKCE

**Why PKCE?**
- Prevents authorization code interception attacks
- Required for mobile and public clients
- Recommended for all OAuth implementations

**Implementation Pattern:**

```typescript
class SecureOAuthFlow {
  async initiateAuthFlow(config: OAuthConfig): Promise<OAuthTokens> {
    // 1. Generate code verifier (random string)
    const codeVerifier = this.generateCodeVerifier();
    
    // 2. Generate code challenge (SHA256 hash of verifier)
    const codeChallenge = await this.generateCodeChallenge(codeVerifier);
    
    // 3. Generate state (CSRF protection)
    const state = this.generateState();
    
    // 4. Store verifier and state securely (temporary)
    await this.secureStorage.storeTemporary('oauth_verifier', codeVerifier);
    await this.secureStorage.storeTemporary('oauth_state', state);
    
    // 5. Build authorization URL
    const authUrl = this.buildAuthUrl(config, state, codeChallenge);
    
    // 6. Open browser/WebView
    const authCode = await this.openAuthFlow(authUrl);
    
    // 7. Validate state (CSRF check)
    const storedState = await this.secureStorage.retrieveTemporary('oauth_state');
    if (authCode.state !== storedState) {
      throw new SecurityError('State mismatch - possible CSRF attack');
    }
    
    // 8. Exchange code for tokens (with verifier)
    const storedVerifier = await this.secureStorage.retrieveTemporary('oauth_verifier');
    const tokens = await this.exchangeCodeForTokens(config, authCode.code, storedVerifier);
    
    // 9. Clean up temporary data
    await this.secureStorage.clearTemporary('oauth_verifier');
    await this.secureStorage.clearTemporary('oauth_state');
    
    // 10. Store tokens securely
    await this.secureStorage.storeTokens(config.provider, tokens);
    
    return tokens;
  }
  
  private generateCodeVerifier(): string {
    // RFC 7636: 43-128 characters, URL-safe base64
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return base64URLEncode(array);
  }
  
  private async generateCodeChallenge(verifier: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const hash = await crypto.subtle.digest('SHA-256', data);
    return base64URLEncode(new Uint8Array(hash));
  }
  
  private generateState(): string {
    // Random state for CSRF protection
    return crypto.randomUUID();
  }
}
```

### Common OAuth Vulnerabilities

#### 1. Authorization Code Interception
**Risk**: Attacker intercepts authorization code  
**Mitigation**: Use PKCE (code verifier/challenge)

#### 2. CSRF Attacks
**Risk**: Attacker uses victim's session  
**Mitigation**: Use state parameter, validate on callback

#### 3. Token Storage Vulnerabilities
**Risk**: Tokens stored insecurely, accessible to other apps  
**Mitigation**: Use platform secure storage (Keychain, Keystore)

#### 4. Token Leakage
**Risk**: Tokens exposed in logs, URLs, or error messages  
**Mitigation**: Never log tokens, use secure channels only

#### 5. Redirect URI Manipulation
**Risk**: Attacker redirects to malicious URI  
**Mitigation**: Validate redirect URI, use exact match

---

## Token Management Strategies

### Token Refresh Pattern

```typescript
class TokenManager {
  private tokenStorage: SecureTokenStorage;
  
  async getValidAccessToken(provider: string): Promise<string> {
    const tokens = await this.tokenStorage.getTokens(provider);
    if (!tokens) {
      throw new AuthenticationRequiredError('No tokens found');
    }
    
    // Check if token expires soon (within 5 minutes)
    const expiresSoon = tokens.expiresAt.getTime() - Date.now() < 5 * 60 * 1000;
    
    if (expiresSoon && tokens.refreshToken) {
      try {
        // Refresh token automatically
        const newTokens = await this.refreshAccessToken(provider);
        return newTokens.accessToken;
      } catch (error) {
        // Refresh failed, need re-authentication
        await this.tokenStorage.clearTokens(provider);
        throw new AuthenticationRequiredError('Token refresh failed', error);
      }
    }
    
    return tokens.accessToken;
  }
  
  async refreshAccessToken(provider: string): Promise<OAuthTokens> {
    const config = this.getOAuthConfig(provider);
    const storedTokens = await this.tokenStorage.getTokens(provider);
    
    if (!storedTokens?.refreshToken) {
      throw new Error('No refresh token available');
    }
    
    const response = await fetch(config.tokenEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: new URLSearchParams({
        grant_type: 'refresh_token',
        refresh_token: storedTokens.refreshToken,
        client_id: config.clientId,
        ...(config.clientSecret && { client_secret: config.clientSecret })
      })
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new OAuthError('Token refresh failed', error);
    }
    
    const data = await response.json();
    const newTokens: OAuthTokens = {
      accessToken: data.access_token,
      refreshToken: data.refresh_token || storedTokens.refreshToken,
      expiresIn: data.expires_in,
      tokenType: data.token_type,
      expiresAt: new Date(Date.now() + data.expires_in * 1000)
    };
    
    await this.tokenStorage.storeTokens(provider, newTokens);
    return newTokens;
  }
}
```

### Token Revocation

```typescript
async revokeAccess(provider: string): Promise<void> {
  const config = this.getOAuthConfig(provider);
  const tokens = await this.tokenStorage.getTokens(provider);
  
  if (!tokens || !config.revocationEndpoint) {
    return;
  }
  
  try {
    await fetch(config.revocationEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: new URLSearchParams({
        token: tokens.accessToken,
        token_type_hint: 'access_token',
        client_id: config.clientId
      })
    });
  } catch (error) {
    // Log error but continue with local cleanup
    console.error('Token revocation failed:', error);
  } finally {
    // Always clear local tokens
    await this.tokenStorage.clearTokens(provider);
  }
}
```

---

## Secure Storage Implementation

### iOS Keychain Storage

```typescript
import KeychainSwift from 'keychain-swift';

class IOSSecureStorage implements SecureTokenStorage {
  private keychain: KeychainSwift;
  
  constructor() {
    this.keychain = new KeychainSwift();
    this.keychain.synchronizable = false; // Don't sync to iCloud
  }
  
  async storeTokens(provider: string, tokens: OAuthTokens): Promise<void> {
    const key = `oauth_tokens_${provider}`;
    const data = JSON.stringify(tokens);
    
    // Keychain automatically encrypts
    const success = this.keychain.set(data, key, {
      accessible: KeychainSwiftAccessible.WHEN_UNLOCKED_THIS_DEVICE_ONLY
    });
    
    if (!success) {
      throw new SecurityError('Failed to store tokens in Keychain');
    }
    
    // Also store access token in Secure Enclave for additional security
    await this.storeInSecureEnclave(provider, tokens.accessToken);
  }
  
  async getTokens(provider: string): Promise<OAuthTokens | null> {
    const key = `oauth_tokens_${provider}`;
    const data = this.keychain.get(key);
    
    if (!data) return null;
    
    try {
      return JSON.parse(data);
    } catch (error) {
      throw new SecurityError('Failed to parse stored tokens');
    }
  }
  
  async clearTokens(provider: string): Promise<void> {
    const key = `oauth_tokens_${provider}`;
    this.keychain.delete(key);
    await this.clearSecureEnclave(provider);
  }
  
  private async storeInSecureEnclave(provider: string, token: string): Promise<void> {
    // Use iOS Secure Enclave for additional security
    // Implementation uses SecItemAdd with kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    // and kSecAttrAccessControl with biometric authentication
  }
}
```

**Keychain Best Practices:**
- Use `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` for maximum security
- Don't sync to iCloud (`synchronizable = false`)
- Use Secure Enclave for sensitive tokens
- Clear tokens on app uninstall

---

### Android Keystore Storage

```typescript
import * as KeyStore from 'react-native-keychain';

class AndroidSecureStorage implements SecureTokenStorage {
  async storeTokens(provider: string, tokens: OAuthTokens): Promise<void> {
    const key = `oauth_tokens_${provider}`;
    const data = JSON.stringify(tokens);
    
    await KeyStore.setGenericPassword(key, data, {
      accessControl: KeyStore.ACCESS_CONTROL.BIOMETRY_ANY_OR_DEVICE_PASSCODE,
      accessible: KeyStore.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
      service: 'life_os_oauth',
      storage: KeyStore.STORAGE_TYPE.RSA // Use hardware-backed storage
    });
  }
  
  async getTokens(provider: string): Promise<OAuthTokens | null> {
    const key = `oauth_tokens_${provider}`;
    const credentials = await KeyStore.getGenericPassword({
      service: 'life_os_oauth',
      authenticationPrompt: {
        title: 'Authenticate to access tokens',
        subtitle: 'Life OS needs your authentication',
        description: 'Use biometrics to unlock',
        cancel: 'Cancel'
      }
    });
    
    if (!credentials || credentials.username !== key) {
      return null;
    }
    
    try {
      return JSON.parse(credentials.password);
    } catch (error) {
      throw new SecurityError('Failed to parse stored tokens');
    }
  }
  
  async clearTokens(provider: string): Promise<void> {
    await KeyStore.resetGenericPassword({
      service: 'life_os_oauth'
    });
  }
}
```

**Keystore Best Practices:**
- Use hardware-backed storage (`STORAGE_TYPE.RSA`)
- Require biometric authentication for access
- Use `WHEN_UNLOCKED_THIS_DEVICE_ONLY` for maximum security
- Clear tokens on app uninstall

---

### Web Encryption Storage

```typescript
class WebSecureStorage implements SecureTokenStorage {
  private encryptionKey: CryptoKey;
  
  async initialize(): Promise<void> {
    this.encryptionKey = await this.getOrCreateEncryptionKey();
  }
  
  async storeTokens(provider: string, tokens: OAuthTokens): Promise<void> {
    const key = `oauth_tokens_${provider}`;
    const encrypted = await this.encrypt(JSON.stringify(tokens));
    
    localStorage.setItem(key, encrypted);
  }
  
  async getTokens(provider: string): Promise<OAuthTokens | null> {
    const key = `oauth_tokens_${provider}`;
    const encrypted = localStorage.getItem(key);
    
    if (!encrypted) return null;
    
    try {
      const decrypted = await this.decrypt(encrypted);
      return JSON.parse(decrypted);
    } catch (error) {
      throw new SecurityError('Failed to decrypt tokens');
    }
  }
  
  async clearTokens(provider: string): Promise<void> {
    const key = `oauth_tokens_${provider}`;
    localStorage.removeItem(key);
  }
  
  private async encrypt(data: string): Promise<string> {
    const encoder = new TextEncoder();
    const dataBuffer = encoder.encode(data);
    
    // Generate random IV for each encryption
    const iv = crypto.getRandomValues(new Uint8Array(12));
    
    const encrypted = await crypto.subtle.encrypt(
      { name: 'AES-GCM', iv },
      this.encryptionKey,
      dataBuffer
    );
    
    // Combine IV and encrypted data
    const combined = new Uint8Array(iv.length + encrypted.byteLength);
    combined.set(iv);
    combined.set(new Uint8Array(encrypted), iv.length);
    
    return base64Encode(combined);
  }
  
  private async decrypt(encrypted: string): Promise<string> {
    const combined = base64Decode(encrypted);
    const iv = combined.slice(0, 12);
    const data = combined.slice(12);
    
    const decrypted = await crypto.subtle.decrypt(
      { name: 'AES-GCM', iv },
      this.encryptionKey,
      data
    );
    
    const decoder = new TextDecoder();
    return decoder.decode(decrypted);
  }
  
  private async getOrCreateEncryptionKey(): Promise<CryptoKey> {
    const keyName = 'life_os_encryption_key';
    const storedKey = localStorage.getItem(keyName);
    
    if (storedKey) {
      const keyData = base64Decode(storedKey);
      return crypto.subtle.importKey(
        'raw',
        keyData,
        { name: 'AES-GCM' },
        false,
        ['encrypt', 'decrypt']
      );
    }
    
    // Generate new key
    const key = await crypto.subtle.generateKey(
      { name: 'AES-GCM', length: 256 },
      true,
      ['encrypt', 'decrypt']
    );
    
    // Export and store (encrypted with user password in production)
    const exported = await crypto.subtle.exportKey('raw', key);
    localStorage.setItem(keyName, base64Encode(new Uint8Array(exported)));
    
    return key;
  }
}
```

**Web Storage Best Practices:**
- Use AES-GCM encryption (authenticated encryption)
- Generate random IV for each encryption
- Store encryption key securely (consider user password encryption)
- Use HTTPS only
- Consider IndexedDB for larger data

---

## Security Audit Checklist

### Authentication & Authorization
- [ ] PKCE implemented for all OAuth flows
- [ ] State parameter used and validated (CSRF protection)
- [ ] Redirect URI validated (exact match)
- [ ] Tokens never logged or exposed in URLs
- [ ] Token refresh implemented with automatic retry
- [ ] Token revocation implemented
- [ ] Multi-factor authentication considered (future)

### Token Storage
- [ ] Tokens stored in platform secure storage (Keychain/Keystore)
- [ ] Tokens encrypted at rest (web)
- [ ] Tokens never stored in plain text
- [ ] Refresh tokens stored securely
- [ ] Tokens cleared on logout/uninstall
- [ ] Token expiration checked before use

### Data Transmission
- [ ] All API calls use HTTPS/TLS 1.2+
- [ ] Certificate pinning considered (for sensitive APIs)
- [ ] No sensitive data in query parameters
- [ ] Request/response validation implemented
- [ ] Error messages don't expose sensitive information

### API Security
- [ ] API keys stored securely (not in code)
- [ ] Rate limiting implemented
- [ ] Input validation on all API calls
- [ ] Output sanitization for user-facing data
- [ ] Error handling doesn't leak information

### Platform Security
- [ ] iOS: Keychain access controls configured
- [ ] Android: Keystore hardware-backed storage used
- [ ] Web: Encryption key management secure
- [ ] Permissions requested minimally
- [ ] Permission denial handled gracefully

### Code Security
- [ ] No hardcoded secrets or API keys
- [ ] Code obfuscation considered (for sensitive logic)
- [ ] Dependencies regularly updated
- [ ] Security vulnerabilities scanned
- [ ] Secrets management system in place

### Privacy & Compliance
- [ ] Privacy policy includes all data collection
- [ ] User consent obtained for sensitive data
- [ ] Data minimization practiced
- [ ] Data retention policies implemented
- [ ] User data deletion supported

---

## Security Testing

### Penetration Testing Checklist
- [ ] OAuth flow tested for vulnerabilities
- [ ] Token storage tested for extraction
- [ ] API endpoints tested for injection attacks
- [ ] Rate limiting tested for bypass
- [ ] Error handling tested for information leakage
- [ ] Network traffic analyzed for sensitive data

### Security Tools
- **Static Analysis**: ESLint security plugins, SonarQube
- **Dependency Scanning**: npm audit, Snyk, Dependabot
- **Dynamic Analysis**: OWASP ZAP, Burp Suite
- **Mobile Security**: MobSF, Frida
- **Network Analysis**: Wireshark, mitmproxy

---

## Incident Response Plan

### Security Incident Types
1. **Token Compromise**: Revoke tokens immediately
2. **API Key Leakage**: Rotate API keys
3. **Data Breach**: Notify users, investigate, fix
4. **OAuth Attack**: Revoke tokens, investigate, update flow

### Response Procedures
1. **Detect**: Monitor for suspicious activity
2. **Contain**: Revoke access, disable affected features
3. **Investigate**: Analyze logs, identify root cause
4. **Remediate**: Fix vulnerability, update security
5. **Notify**: Inform users if data compromised
6. **Document**: Record incident and resolution

---

## Security Best Practices Summary

### Do's ✅
- Use PKCE for all OAuth flows
- Store tokens in platform secure storage
- Encrypt sensitive data at rest and in transit
- Validate all inputs and sanitize outputs
- Implement rate limiting
- Use HTTPS for all API calls
- Clear tokens on logout
- Handle errors gracefully without exposing details

### Don'ts ❌
- Don't store tokens in plain text
- Don't log tokens or sensitive data
- Don't expose tokens in URLs or error messages
- Don't hardcode API keys or secrets
- Don't skip input validation
- Don't ignore security warnings
- Don't use deprecated security methods
- Don't store sensitive data unnecessarily

---

## Compliance Considerations

### GDPR Requirements
- **Right to Access**: Users can request their data
- **Right to Deletion**: Users can request data deletion
- **Data Portability**: Users can export their data
- **Consent**: Explicit consent for sensitive data
- **Privacy by Design**: Security built-in from start

### Platform Requirements
- **Apple**: Privacy Nutrition Labels, usage descriptions
- **Google**: Data Safety form, permission justification
- **Both**: Privacy policy required, data handling disclosure

---

## Next Steps

1. **Week 1**: Implement secure storage for all platforms
2. **Week 2**: Implement PKCE for OAuth flows
3. **Week 3**: Security audit and penetration testing
4. **Week 4**: Address security findings
5. **Week 5**: Final security review
6. **Week 6**: Security documentation completion

---

## Resources

- [OWASP Mobile Security](https://owasp.org/www-project-mobile-security/)
- [OAuth 2.0 Security Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [Apple Security Guidelines](https://developer.apple.com/security/)
- [Android Security Best Practices](https://developer.android.com/training/best-security)

---

## Notes

- Security is an ongoing process, not a one-time task
- Regular security audits recommended (quarterly)
- Stay updated with security advisories
- Monitor for new vulnerabilities
- Update dependencies regularly
- Review and update security practices as threats evolve

