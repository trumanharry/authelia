# Beta SSO Implementation Guide: NocoBase & HumHub with Authelia on Elestio

**Implementation Time: 2-3 hours**  
**Target: Beta testing environment with basic SSO functionality**  
**Date: November 13, 2025**

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Elestio account with payment method
- [ ] Domain name with DNS management access
- [ ] NocoBase instance running on Elestio (with OIDC plugin installed)
- [ ] HumHub instance running on Elestio (with Authelia module installed)
- [ ] Admin access to both NocoBase and HumHub
- [ ] Text editor for configuration files

## Phase 1: Authelia Deployment (30 minutes)

### Step 1.1: Deploy Authelia on Elestio (10 minutes)

1. **Login to Elestio Dashboard**
   - Go to https://elest.io
   - Login to your account

2. **Create New Service**
   - Click "Create Service"
   - Search for "Authelia"
   - Select "Authelia" from the catalog

3. **Configure Deployment**
   - **Service Name**: `authelia-sso` (or your preferred name)
   - **Cloud Provider**: Choose same as your other services
   - **Region**: Choose same region as NocoBase/HumHub
   - **Instance Size**: Select "Small" (1 vCPU, 1GB RAM) - sufficient for beta
   - **Project**: Use same project as NocoBase and HumHub

4. **Deploy Service**
   - Click "Create Service"
   - Wait 2-3 minutes for deployment
   - Note the temporary URL (format: `xxxxxxxx-uX.vm.elestio.app`)

### Step 1.2: Configure Custom Domain (10 minutes)

1. **Add Custom Domain in Elestio**
   - Go to your Authelia service dashboard
   - Click "Security" → "Domain Management"
   - Click "Add Custom Domain"
   - Enter: `auth.yourdomain.com` (replace with your actual domain)

2. **Update DNS Records**
   - Go to your domain registrar/DNS provider
   - Create a CNAME record:
     - **Name**: `auth`
     - **Value**: `xxxxxxxx-uX.vm.elestio.app` (your Elestio URL)
     - **TTL**: 300 (5 minutes)

3. **Verify SSL Certificate**
   - Wait 5-15 minutes for DNS propagation
   - SSL certificate will generate automatically
   - Verify by visiting `https://auth.yourdomain.com`

### Step 1.3: Access Authelia Configuration (10 minutes)

1. **Get SSH Access**
   - In Elestio dashboard, click "Tools" → "VS Code"
   - Or use "SSH" for terminal access
   - Credentials are in the "Overview" tab

2. **Locate Configuration Files**
   ```bash
   cd /config
   ls -la
   ```
   
3. **Backup Default Configuration**
   ```bash
   cp configuration.yml configuration.yml.backup
   cp users_database.yml users_database.yml.backup
   ```

## Phase 2: Authelia OIDC Configuration (45 minutes)

### Step 2.1: Generate Required Secrets (15 minutes)

1. **Access Authelia Container**
   ```bash
   # In VS Code terminal or SSH
   docker exec -it authelia authelia --version
   ```

2. **Generate HMAC Secret**
   ```bash
   docker exec -it authelia authelia crypto rand --length 64 --charset alphanumeric
   ```
   **Save this output** - you'll need it for configuration

3. **Generate RSA Key Pair**
   ```bash
   docker exec -it authelia authelia crypto certificate rsa generate \
     --common-name auth.yourdomain.com \
     --directory /config \
     --file.private-key rsa.key \
     --file.certificate rsa.cert
   ```

4. **Generate Client Secrets**
   ```bash
   # For NocoBase
   docker exec -it authelia authelia crypto rand --length 64 --charset alphanumeric
   
   # For HumHub  
   docker exec -it authelia authelia crypto rand --length 64 --charset alphanumeric
   ```
   **Save both outputs** - these are your client secrets

5. **Hash Client Secrets**
   ```bash
   # Hash NocoBase client secret
   docker exec -it authelia authelia crypto hash generate pbkdf2 --password "YOUR_NOCOBASE_CLIENT_SECRET"
   
   # Hash HumHub client secret
   docker exec -it authelia authelia crypto hash generate pbkdf2 --password "YOUR_HUMHUB_CLIENT_SECRET"
   ```
   **Save the hashed outputs** for Authelia configuration

### Step 2.2: Configure Identity Provider (15 minutes)

1. **Edit Main Configuration File**
   ```bash
   # In VS Code or terminal
   nano /config/configuration.yml
   ```

2. **Add OIDC Identity Provider Section**
   Add this section to your configuration.yml:

   ```yaml
   identity_providers:
     oidc:
       hmac_secret: 'YOUR_GENERATED_HMAC_SECRET_HERE'
       
       lifespans:
         access_token: '1h'
         refresh_token: '90m'
         id_token: '1h'
         authorize_code: '1m'
       
       cors:
         endpoints:
           - 'authorization'
           - 'token'
           - 'revocation'
           - 'introspection'
           - 'userinfo'
         allowed_origins_from_client_redirect_uris: true
       
       jwks:
         - key_id: 'main-signing-key'
           algorithm: 'RS256'
           use: 'sig'
           key: |
             -----BEGIN PRIVATE KEY-----
             [PASTE YOUR RSA PRIVATE KEY CONTENT HERE]
             -----END PRIVATE KEY-----
   ```

3. **Get RSA Private Key Content**
   ```bash
   cat /config/rsa.key
   ```
   Copy the entire key (including BEGIN/END lines) and paste into the configuration

### Step 2.3: Configure OIDC Clients (15 minutes)

1. **Add Clients Section to configuration.yml**
   
   Continue adding to your identity_providers.oidc section:

   ```yaml
       clients:
         # NocoBase Client
         - client_id: 'nocobase-beta'
           client_name: 'NocoBase Beta Application'
           client_secret: 'YOUR_HASHED_NOCOBASE_SECRET_HERE'
           public: false
           authorization_policy: 'one_factor'
           consent_mode: 'auto'
           
           redirect_uris:
             - 'https://nocobase.yourdomain.com/api/auth:redirect?authenticator=oidc-plus'
           
           scopes:
             - 'openid'
             - 'profile'
             - 'email'
             - 'groups'
           
           grant_types:
             - 'authorization_code'
             - 'refresh_token'
           
           response_types:
             - 'code'
           
           userinfo_signed_response_alg: 'none'
         
         # HumHub Client
         - client_id: 'humhub-beta'
           client_name: 'HumHub Beta Social Network'
           client_secret: 'YOUR_HASHED_HUMHUB_SECRET_HERE'
           public: false
           authorization_policy: 'one_factor'
           consent_mode: 'auto'
           
           redirect_uris:
             - 'https://humhub.yourdomain.com/user/auth/external?authclient=authelia'
           
           scopes:
             - 'openid'
             - 'profile'
             - 'email'
             - 'groups'
           
           grant_types:
             - 'authorization_code'
             - 'refresh_token'
           
           response_types:
             - 'code'
   ```

   **Important**: Replace `yourdomain.com` with your actual domain!

## Phase 3: User Configuration (15 minutes)

### Step 3.1: Configure Test Users

1. **Edit Users Database**
   ```bash
   nano /config/users_database.yml
   ```

2. **Add Test Users**
   Replace the default content with:

   ```yaml
   users:
     admin:
       displayname: "Admin User"
       password: "$6$rounds=50000$BpLnfgDsc2WD8F2q$Hx4ZZoZoU35G8dq5D8x8d5V5m3K3P3R3T3Y3Z3A3B3C3E3F3H3J3K3L3M3N3P3Q3R3S3T3U3V3W3X3Y3Z3"
       email: admin@yourdomain.com
       groups:
         - admins
         - dev
     
     testuser1:
       displayname: "Test User 1"
       password: "$6$rounds=50000$BpLnfgDsc2WD8F2q$Hx4ZZoZoU35G8dq5D8x8d5V5m3K3P3R3T3Y3Z3A3B3C3E3F3H3J3K3L3M3N3P3Q3R3S3T3U3V3W3X3Y3Z3"
       email: test1@yourdomain.com
       groups:
         - users
     
     testuser2:
       displayname: "Test User 2"  
       password: "$6$rounds=50000$BpLnfgDsc2WD8F2q$Hx4ZZoZoU35G8dq5D8x8d5V5m3K3P3R3T3Y3Z3A3B3C3E3F3H3J3K3L3M3N3P3Q3R3S3T3U3V3W3X3Y3Z3"
       email: test2@yourdomain.com
       groups:
         - users
   ```

3. **Generate Real Password Hashes**
   ```bash
   # For each user, generate a password hash
   docker exec -it authelia authelia crypto hash generate argon2 --password "admin123"
   docker exec -it authelia authelia crypto hash generate argon2 --password "test123"
   ```
   Replace the placeholder hashes with the generated ones

### Step 3.2: Restart Authelia Service (5 minutes)

1. **Save All Files**
   - Ensure configuration.yml is saved
   - Ensure users_database.yml is saved

2. **Restart Container**
   ```bash
   docker restart authelia
   ```

3. **Verify Service**
   ```bash
   docker logs authelia --tail 50
   ```
   Check for any error messages

4. **Test Authelia Interface**
   - Visit: `https://auth.yourdomain.com`
   - Should show Authelia login page
   - Try logging in with admin/admin123

## Phase 4: NocoBase Integration (30 minutes)

### Step 4.1: Configure NocoBase OIDC Plugin (20 minutes)

1. **Access NocoBase Admin Panel**
   - Go to `https://nocobase.yourdomain.com`
   - Login as administrator

2. **Navigate to Authentication Settings**
   - Go to "Settings" → "Authentication"
   - Find "OIDC Plus" plugin
   - Click "Configure" or "Settings"

3. **Configure OIDC Settings**
   Fill in these values:

   ```
   Provider Name: Authelia SSO
   Client ID: nocobase-beta
   Client Secret: [YOUR_PLAIN_NOCOBASE_CLIENT_SECRET]
   Discovery URL: https://auth.yourdomain.com/.well-known/openid-configuration
   Scopes: openid profile email groups
   
   Authorization URL: https://auth.yourdomain.com/api/oidc/authorization
   Token URL: https://auth.yourdomain.com/api/oidc/token
   User Info URL: https://auth.yourdomain.com/api/oidc/userinfo
   
   Attribute Mapping:
   - Username: sub
   - Display Name: name
   - Email: email
   - Groups: groups
   ```

4. **Enable the Authentication Method**
   - Save the configuration
   - Enable the OIDC authentication method
   - Set it as available for new user registration

### Step 4.2: Test NocoBase Integration (10 minutes)

1. **Logout from NocoBase**
   - Logout from admin account

2. **Test SSO Login**
   - Look for "Authelia SSO" or "OIDC" login button
   - Click the SSO login option
   - Should redirect to Authelia
   - Login with test user credentials
   - Should redirect back to NocoBase

3. **Verify User Creation**
   - Check if user account was created automatically
   - Verify user attributes are mapped correctly

## Phase 5: HumHub Integration (30 minutes)

### Step 5.1: Configure HumHub Authelia Module (20 minutes)

1. **Access HumHub Admin Panel**
   - Go to `https://humhub.yourdomain.com`
   - Login as administrator

2. **Navigate to Module Configuration**
   - Go to "Administration" → "Modules"
   - Find "Authelia Auth" module
   - Click "Configure"

3. **Configure Authelia Settings**
   Fill in these values:

   ```
   Authelia Base URL: https://auth.yourdomain.com
   Client ID: humhub-beta
   Client Secret: [YOUR_PLAIN_HUMHUB_CLIENT_SECRET]
   Scopes: openid profile email groups
   
   Button Text: "Login with Authelia"
   Button Icon: fa fa-sign-in
   
   User Attribute Mapping:
   - Username: sub
   - Display Name: name  
   - Email: email
   ```

4. **Save and Enable**
   - Save the configuration
   - Enable the authentication method
   - Check "Allow new user registration via Authelia"

### Step 5.2: Test HumHub Integration (10 minutes)

1. **Logout from HumHub**
   - Logout from admin account

2. **Test SSO Login**
   - Look for "Login with Authelia" button on login page
   - Click the Authelia login button
   - Should redirect to Authelia
   - Login with test user credentials
   - Should redirect back to HumHub

3. **Verify User Creation**
   - Check if user account was created automatically
   - Verify user profile information

## Phase 6: Testing & Validation (20 minutes)

### Step 6.1: Single Sign-On Flow Test (10 minutes)

1. **Complete SSO Test Sequence**
   - Login to NocoBase via Authelia
   - In new browser tab, go to HumHub
   - Should be automatically logged in (SSO)
   - Verify same user identity in both applications

2. **Cross-Application Navigation**
   - Navigate between NocoBase and HumHub
   - Verify persistent authentication state
   - Test with different test users

### Step 6.2: User Experience Validation (10 minutes)

1. **Test User Registration Flow**
   - Use incognito browser
   - Try SSO login with new test user
   - Verify automatic account creation in both apps

2. **Test Logout Flow**
   - Logout from one application
   - Check if logout propagates (may require additional configuration)

3. **Error Handling Test**
   - Try invalid credentials
   - Verify proper error messages
   - Test redirect handling

## Troubleshooting Common Issues

### Issue 1: "Invalid Redirect URI"
**Symptoms**: Error during OIDC redirect
**Solution**: 
1. Check redirect URIs in Authelia client configuration
2. Ensure URLs exactly match your domain
3. Verify HTTPS is used consistently

### Issue 2: "Client Authentication Failed"
**Symptoms**: 401 error during token exchange
**Solution**:
1. Verify client secrets match (plain text in apps, hashed in Authelia)
2. Check client IDs are identical
3. Ensure client is enabled in Authelia

### Issue 3: Users Not Created Automatically
**Symptoms**: Authentication succeeds but no user account created
**Solution**:
1. Check attribute mapping configuration
2. Verify scopes include required fields (email, profile)
3. Enable automatic user registration in app settings

### Issue 4: SSL Certificate Errors
**Symptoms**: "Connection not secure" warnings
**Solution**:
1. Wait for DNS propagation (up to 1 hour)
2. Check CNAME record is correct
3. Restart Authelia container if needed

## Beta Testing Checklist

Before inviting test users:

- [ ] All three services accessible via HTTPS
- [ ] SSO login works for NocoBase
- [ ] SSO login works for HumHub  
- [ ] Test user accounts created automatically
- [ ] User attributes mapped correctly
- [ ] Error messages are user-friendly
- [ ] Mobile browser compatibility verified
- [ ] Backup admin access available (local accounts)

## Security Notes for Beta

**Current Configuration is for BETA TESTING ONLY**

For production deployment, implement:

1. **Stronger Authentication**
   - Change `authorization_policy` from 'one_factor' to 'two_factor'
   - Enable MFA for admin accounts

2. **Enhanced Security**
   - Rotate all client secrets
   - Use stronger password policies
   - Enable rate limiting
   - Implement proper backup strategies

3. **Monitoring**
   - Enable audit logging
   - Monitor authentication attempts
   - Set up health checks

## Next Steps

After successful beta testing:

1. **Gather User Feedback**
   - Authentication flow usability
   - Performance issues
   - Missing features

2. **Plan Production Migration**
   - Enhanced security configuration
   - Backup and recovery procedures
   - Monitoring and alerting setup
   - User training documentation

3. **Scale Preparation**
   - Performance testing with more users
   - Database optimization
   - Load balancing considerations

## Support Resources

- **Authelia Documentation**: https://www.authelia.com/configuration/
- **Elestio Support**: Available through dashboard
- **NocoBase OIDC Plugin**: Check plugin documentation
- **HumHub Authelia Module**: GitHub repository for issues

## Configuration Summary

**Total Implementation Time**: 2-3 hours  
**Monthly Hosting Cost**: ~$30-40 (3 small Elestio instances)  
**Plugin Costs**: $0 (using free community plugins)  
**Maintenance Effort**: Minimal for beta phase  

Your SSO system should now be ready for beta testing with automatic user provisioning and centralized authentication across both applications.
