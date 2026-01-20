# Instructions de Configuration Supabase Cloud

**Story:** 1.1 - Project Foundation & Infrastructure Setup
**Composant:** Supabase Cloud Authentication
**Temps estimé:** 30 minutes

---

## 📋 Objectif

Configurer un projet Supabase Cloud (tier gratuit) avec:
- ✅ Authentification Email/Password
- ✅ Google OAuth
- ✅ Apple Sign-In
- ✅ SMTP pour emails de récupération de mot de passe

---

## Étape 1: Créer le Projet Supabase

### 1.1 Accéder au Dashboard

1. Ouvrir le navigateur et aller sur: https://supabase.com/dashboard
2. Se connecter ou créer un compte Supabase (gratuit)

### 1.2 Créer un Nouveau Projet

1. Cliquer sur le bouton **"New Project"**
2. Remplir les informations:
   - **Organization:** Sélectionner ou créer une organisation
   - **Name:** `pensine`
   - **Database Password:** Générer un mot de passe fort (sauvegarder dans un gestionnaire de mots de passe)
   - **Region:** `Europe (Frankfurt)` - **IMPORTANT pour conformité RGPD**
   - **Pricing Plan:** Free (50,000 utilisateurs actifs mensuels)

3. Cliquer sur **"Create new project"**
4. Attendre ~2 minutes pour le provisioning

---

## Étape 2: Configurer l'Authentification Email/Password

### 2.1 Activer le Provider Email

1. Dans le dashboard, aller à: **Authentication → Providers**
2. Trouver **Email** dans la liste
3. Activer les options:
   - ✅ **Enable Email provider**
   - ✅ **Confirm email** (activation par email obligatoire)
   - ✅ **Secure email change** (confirmation pour changement d'email)

4. Cliquer sur **"Save"**

### 2.2 Personnaliser les Templates d'Email (Optionnel pour MVP)

Pour l'instant, garder les templates par défaut. Ils seront personnalisés dans une story ultérieure.

---

## Étape 3: Configurer Google OAuth

### 3.1 Créer les Credentials Google Cloud

1. Aller sur: https://console.cloud.google.com/
2. Créer un nouveau projet ou sélectionner un projet existant:
   - **Nom du projet:** `Pensine`

3. Activer l'API Google+ :
   - Menu hamburger → **APIs & Services → Library**
   - Rechercher: `Google+ API`
   - Cliquer sur **"Enable"**

4. Créer les OAuth 2.0 Credentials:
   - Menu hamburger → **APIs & Services → Credentials**
   - Cliquer sur **"+ CREATE CREDENTIALS" → OAuth client ID**
   - Si demandé, configurer l'écran de consentement OAuth:
     - Type: **External**
     - App name: `Pensine`
     - User support email: votre email
     - Developer contact: votre email
     - Cliquer **"Save and Continue"** jusqu'à la fin

5. Créer OAuth Client ID:
   - **Application type:** Web application
   - **Name:** `Pensine Web Client`
   - **Authorized redirect URIs:** Ajouter l'URL de callback Supabase (voir ci-dessous)

### 3.2 Récupérer l'URL de Callback Supabase

1. Dans Supabase Dashboard → **Authentication → Providers → Google**
2. Copier l'URL affichée dans **"Callback URL (for OAuth)"**:
   ```
   https://xxxxx.supabase.co/auth/v1/callback
   ```

3. Retourner dans Google Cloud Console
4. Dans **"Authorized redirect URIs"**, coller l'URL de callback
5. Cliquer **"CREATE"**

### 3.3 Configurer Google OAuth dans Supabase

1. Dans Google Cloud Console, copier:
   - **Client ID:** `123456789-abcdefg.apps.googleusercontent.com`
   - **Client Secret:** `GOCSPX-xxxxxxxxxxxxxxx`

2. Dans Supabase Dashboard → **Authentication → Providers → Google**:
   - Coller **Client ID** dans le champ correspondant
   - Coller **Client Secret** dans le champ correspondant
   - ✅ **Enable Google provider**

3. Cliquer **"Save"**

---

## Étape 4: Configurer Apple Sign-In

### 4.1 Prérequis

- Avoir un **Apple Developer Account** (99€/an)
- Accès au **Apple Developer Portal**

### 4.2 Créer un App ID

1. Aller sur: https://developer.apple.com/account/resources/identifiers/list
2. Cliquer sur **"+"** pour créer un nouvel identifier
3. Sélectionner **"App IDs"** → Continue
4. Sélectionner **"App"** → Continue
5. Remplir:
   - **Description:** `Pensine App`
   - **Bundle ID:** `com.pensine.app` (explicite)
   - **Capabilities:** Cocher **"Sign in with Apple"**

6. Cliquer **"Continue"** puis **"Register"**

### 4.3 Créer un Service ID

1. Retourner sur: https://developer.apple.com/account/resources/identifiers/list
2. Cliquer sur **"+"** → **"Services IDs"** → Continue
3. Remplir:
   - **Description:** `Pensine Web Service`
   - **Identifier:** `com.pensine.service`

4. Cocher **"Sign in with Apple"** → **"Configure"**
5. Dans la configuration:
   - **Primary App ID:** Sélectionner `com.pensine.app`
   - **Domains and Subdomains:** Ajouter `xxxxx.supabase.co` (votre domaine Supabase)
   - **Return URLs:** Ajouter l'URL de callback Supabase:
     ```
     https://xxxxx.supabase.co/auth/v1/callback
     ```

6. Cliquer **"Save"** puis **"Continue"** puis **"Register"**

### 4.4 Créer une Clé Privée (.p8)

1. Aller sur: https://developer.apple.com/account/resources/authkeys/list
2. Cliquer sur **"+"** pour créer une nouvelle clé
3. Remplir:
   - **Key Name:** `Pensine Sign in with Apple Key`
   - Cocher **"Sign in with Apple"** → **"Configure"**
   - Sélectionner **Primary App ID:** `com.pensine.app`

4. Cliquer **"Save"** puis **"Continue"** puis **"Register"**
5. **Télécharger le fichier .p8** (une seule fois possible !)
6. Noter le **Key ID** affiché (ex: `ABC123DEFG`)

### 4.5 Récupérer le Team ID

1. Aller sur: https://developer.apple.com/account
2. En haut à droite, sous votre nom, noter votre **Team ID** (ex: `XYZ789HIJK`)

### 4.6 Configurer Apple Sign-In dans Supabase

1. Dans Supabase Dashboard → **Authentication → Providers → Apple**
2. Remplir les champs:
   - **Services ID:** `com.pensine.service`
   - **Team ID:** `XYZ789HIJK` (votre Team ID)
   - **Key ID:** `ABC123DEFG` (votre Key ID)
   - **Private Key:** Ouvrir le fichier .p8 et coller tout son contenu

3. ✅ **Enable Apple provider**
4. Cliquer **"Save"**

---

## Étape 5: Configurer SMTP pour les Emails

### 5.1 Configurer un Provider SMTP (Recommandé: SendGrid ou Resend)

**Option A: SendGrid (Gratuit jusqu'à 100 emails/jour)**

1. Créer un compte sur: https://sendgrid.com/
2. Aller dans **Settings → API Keys**
3. Créer une API Key avec permission **"Mail Send"**
4. Noter l'API Key (commence par `SG.`)

**Option B: Resend (Gratuit jusqu'à 3000 emails/mois)**

1. Créer un compte sur: https://resend.com/
2. Créer une API Key
3. Noter l'API Key (commence par `re_`)

### 5.2 Configurer SMTP dans Supabase

1. Dans Supabase Dashboard → **Project Settings → Auth**
2. Scroller jusqu'à **"SMTP Settings"**
3. Activer **"Enable Custom SMTP"**

**Pour SendGrid:**
```
Host: smtp.sendgrid.net
Port: 587
Username: apikey
Password: <votre API Key SendGrid>
Sender email: noreply@pensine.app (ou votre email vérifié)
Sender name: Pensine
```

**Pour Resend:**
```
Host: smtp.resend.com
Port: 587
Username: resend
Password: <votre API Key Resend>
Sender email: noreply@pensine.app (ou votre email vérifié)
Sender name: Pensine
```

4. Cliquer **"Save"**
5. Tester avec le bouton **"Send test email"**

---

## Étape 6: Récupérer les Valeurs de Configuration

### 6.1 Récupérer les Clés API

1. Dans Supabase Dashboard → **Settings → API**
2. Copier les valeurs suivantes:

```bash
# URL du projet Supabase
SUPABASE_URL=https://xxxxx.supabase.co

# Anon Key (clé publique, safe pour le mobile)
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Service Role Key (clé secrète, BACKEND ONLY)
SUPABASE_SERVICE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

⚠️ **IMPORTANT:** Ne JAMAIS exposer `SUPABASE_SERVICE_KEY` dans le code mobile !

### 6.2 Récupérer le JWT Secret

1. Dans Supabase Dashboard → **Settings → API**
2. Scroller jusqu'à **"JWT Settings"**
3. Copier **"JWT Secret"**:

```bash
# JWT Secret (pour validation backend)
JWT_SECRET=your-super-secret-jwt-secret-32-chars-minimum
```

---

## Étape 7: Sauvegarder les Credentials

### 7.1 Créer un Fichier de Credentials Sécurisé

Créer un fichier `supabase-credentials.md` (ne PAS commiter dans Git):

```markdown
# Supabase Credentials - Pensine

**Projet:** pensine
**Région:** Europe (Frankfurt)
**Créé le:** [DATE]

## API Credentials

SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_KEY=eyJhbGc... (BACKEND ONLY)
JWT_SECRET=your-jwt-secret

## OAuth Providers

### Google OAuth
- Client ID: 123456789-abcdefg.apps.googleusercontent.com
- Client Secret: GOCSPX-xxxxxxxxxxxxxxx
- Status: ✅ Configured

### Apple Sign-In
- Services ID: com.pensine.service
- Team ID: XYZ789HIJK
- Key ID: ABC123DEFG
- Private Key: [fichier .p8]
- Status: ✅ Configured

## SMTP Configuration

- Provider: [SendGrid/Resend]
- API Key: [votre API key]
- Sender: noreply@pensine.app
- Status: ✅ Configured
```

### 7.2 Ajouter au .gitignore

Vérifier que le fichier est ignoré:

```bash
echo "supabase-credentials.md" >> .gitignore
```

---

## ✅ Checklist de Validation

Avant de passer à l'étape suivante, vérifier:

- [ ] Projet Supabase créé (région Europe Frankfurt)
- [ ] Email/Password provider activé
- [ ] Google OAuth configuré et testé
- [ ] Apple Sign-In configuré et testé
- [ ] SMTP configuré (test email envoyé avec succès)
- [ ] Credentials sauvegardés de manière sécurisée
- [ ] `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `JWT_SECRET` disponibles

---

## 🔍 Troubleshooting

### Erreur: "Invalid OAuth callback URL"

**Cause:** L'URL de callback n'est pas correctement configurée dans Google Cloud Console ou Apple Developer Portal.

**Solution:**
1. Vérifier que l'URL de callback correspond EXACTEMENT à celle affichée dans Supabase
2. Format attendu: `https://xxxxx.supabase.co/auth/v1/callback`
3. Pas de trailing slash, HTTPS obligatoire

### Erreur: "SMTP connection failed"

**Cause:** Credentials SMTP incorrects ou service non configuré.

**Solution:**
1. Vérifier que l'API Key est valide
2. Pour SendGrid: Username DOIT être `apikey` (littéral)
3. Pour Resend: Username DOIT être `resend` (littéral)
4. Vérifier que le sender email est vérifié dans le service SMTP

### Erreur: "Apple Sign-In - Invalid client"

**Cause:** Service ID ou configuration incorrecte.

**Solution:**
1. Vérifier que le Service ID correspond à celui créé sur Apple Developer
2. Vérifier que le domaine Supabase est bien ajouté dans les domaines autorisés
3. Vérifier que la clé privée .p8 est complète (inclure les lignes BEGIN/END)

---

## 📚 Références

- [Supabase Auth Documentation](https://supabase.com/docs/guides/auth)
- [Google OAuth Setup Guide](https://supabase.com/docs/guides/auth/social-login/auth-google)
- [Apple Sign-In Setup Guide](https://supabase.com/docs/guides/auth/social-login/auth-apple)
- [Supabase SMTP Configuration](https://supabase.com/docs/guides/auth/auth-smtp)

---

**Prochaine étape:** Configurer l'infrastructure homelab (Docker Compose)
