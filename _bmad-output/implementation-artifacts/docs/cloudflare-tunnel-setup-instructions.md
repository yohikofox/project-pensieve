# Instructions de Configuration Cloudflare Tunnel

**Story:** 1.1 - Project Foundation & Infrastructure Setup
**Composant:** Cloudflare Tunnel (Expose Homelab to Internet)
**Temps estimé:** 1-2 heures

---

## 📋 Objectif

Exposer les services homelab de manière sécurisée sur Internet via Cloudflare Tunnel:
- ✅ `api.pensine.app` → NestJS Backend (port 3000)
- ✅ `storage.pensine.app` → MinIO Storage (port 9000)
- ✅ HTTPS automatique via Cloudflare
- ✅ Zero port forwarding (pas d'exposition directe)

---

## Prérequis

### 1. Domaine sur Cloudflare

Vous devez posséder un domaine géré par Cloudflare:
- Si vous avez déjà un domaine: Transférer les DNS vers Cloudflare
- Si vous n'avez pas de domaine: Acheter un domaine (ex: pensine.app sur Namecheap, puis ajouter à Cloudflare)

**Instructions:**
1. Créer un compte gratuit sur: https://dash.cloudflare.com/sign-up
2. Ajouter votre domaine à Cloudflare
3. Mettre à jour les nameservers chez votre registrar:
   ```
   ns1.cloudflare.com
   ns2.cloudflare.com
   ```
4. Attendre la propagation DNS (~24h max, souvent <1h)

### 2. Services Homelab Démarrés

Vérifier que les services Docker sont en cours d'exécution:

```bash
docker-compose ps

# Vérifier que ces services sont "Up (healthy)":
# - pensine-db (PostgreSQL)
# - pensine-queue (RabbitMQ)
# - pensine-storage (MinIO)
```

---

## Étape 1: Installer Cloudflared

### macOS

```bash
# Installation via Homebrew
brew install cloudflared

# Vérifier l'installation
cloudflared --version
# Output: cloudflared version 2024.x.x
```

### Linux (Ubuntu/Debian)

```bash
# Télécharger le package
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Installer
sudo dpkg -i cloudflared-linux-amd64.deb

# Vérifier l'installation
cloudflared --version
```

### Linux (Autres distributions)

```bash
# Télécharger le binaire
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64

# Rendre exécutable
chmod +x cloudflared-linux-amd64

# Déplacer vers /usr/local/bin
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared

# Vérifier
cloudflared --version
```

---

## Étape 2: Authentifier Cloudflare Account

```bash
# Lancer l'authentification
cloudflared tunnel login
```

**Ce qui se passe:**
1. Votre navigateur s'ouvre automatiquement
2. Vous êtes redirigé vers Cloudflare Dashboard
3. **Sélectionnez votre domaine** (ex: `pensine.app`)
4. Cliquez sur **"Authorize"**

**Résultat:**
- Un fichier de certificat est créé: `~/.cloudflared/cert.pem`
- Vous voyez le message: `You have successfully logged in`

---

## Étape 3: Créer le Tunnel

```bash
# Créer un tunnel nommé "pensine"
cloudflared tunnel create pensine
```

**Output attendu:**
```
Tunnel credentials written to: ~/.cloudflared/<TUNNEL-ID>.json
Created tunnel pensine with id <TUNNEL-ID>
```

**Sauvegarder le Tunnel ID:**
```bash
# Copier le TUNNEL-ID depuis l'output
export TUNNEL_ID=<votre-tunnel-id>

# Exemple:
# export TUNNEL_ID=a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

## Étape 4: Configurer les Routes du Tunnel

### 4.1 Créer le Fichier de Configuration

```bash
# Créer le dossier de configuration (si pas déjà existant)
mkdir -p ~/.cloudflared

# Créer le fichier config.yml
cat > ~/.cloudflared/config.yml <<EOF
tunnel: $TUNNEL_ID
credentials-file: /Users/$(whoami)/.cloudflared/${TUNNEL_ID}.json

ingress:
  # API Backend (NestJS)
  - hostname: api.pensine.app
    service: http://localhost:3000
    originRequest:
      noTLSVerify: true

  # Storage (MinIO)
  - hostname: storage.pensine.app
    service: http://localhost:9000
    originRequest:
      noTLSVerify: true

  # Catch-all rule (required)
  - service: http_status:404
EOF
```

**⚠️ IMPORTANT:**
- Remplacer `pensine.app` par votre domaine réel
- Sur Linux, adapter le chemin: `/home/$(whoami)/.cloudflared/${TUNNEL_ID}.json`
- L'ordre des routes est important (catch-all doit être en dernier)

### 4.2 Vérifier la Configuration

```bash
# Vérifier que le fichier est bien créé
cat ~/.cloudflared/config.yml

# Valider la syntaxe
cloudflared tunnel ingress validate
```

**Output attendu:**
```
Validating rules from ~/.cloudflared/config.yml
OK
```

---

## Étape 5: Configurer les DNS

Créer les enregistrements DNS qui pointent vers le tunnel:

```bash
# Configurer le DNS pour l'API
cloudflared tunnel route dns pensine api.pensine.app

# Configurer le DNS pour le Storage
cloudflared tunnel route dns pensine storage.pensine.app
```

**Output attendu (pour chaque commande):**
```
Created CNAME api.pensine.app → <TUNNEL-ID>.cfargotunnel.com
```

**Vérification dans Cloudflare Dashboard:**
1. Aller sur: https://dash.cloudflare.com
2. Sélectionner votre domaine
3. Aller dans **DNS → Records**
4. Vous devriez voir 2 enregistrements CNAME:
   ```
   api      CNAME  <TUNNEL-ID>.cfargotunnel.com
   storage  CNAME  <TUNNEL-ID>.cfargotunnel.com
   ```

---

## Étape 6: Démarrer le Tunnel

### Option A: Mode Foreground (Test)

```bash
# Démarrer le tunnel en mode test
cloudflared tunnel run pensine
```

**Output attendu:**
```
2024-01-19T10:00:00Z INF Starting tunnel tunnelID=<TUNNEL-ID>
2024-01-19T10:00:00Z INF Connection registered connIndex=0
2024-01-19T10:00:00Z INF Connection registered connIndex=1
2024-01-19T10:00:00Z INF Connection registered connIndex=2
2024-01-19T10:00:00Z INF Connection registered connIndex=3
```

**Tester depuis un autre terminal:**
```bash
# Tester MinIO (doit être démarré via docker-compose)
curl https://storage.pensine.app/minio/health/live

# Output attendu: (si MinIO tourne)
# HTTP/1.1 200 OK

# Tester API (sera disponible quand backend sera déployé)
curl https://api.pensine.app/health
# Output attendu: 404 (normal, backend pas encore déployé)
```

### Option B: Mode Service (Production)

Une fois le test réussi, installer comme service système:

#### macOS

```bash
# Installer le service
sudo cloudflared service install

# Démarrer le service
sudo launchctl start com.cloudflare.cloudflared

# Vérifier le status
sudo launchctl list | grep cloudflared
```

#### Linux (systemd)

```bash
# Installer le service
sudo cloudflared service install

# Démarrer le service
sudo systemctl start cloudflared

# Activer au démarrage
sudo systemctl enable cloudflared

# Vérifier le status
sudo systemctl status cloudflared
```

**Output attendu:**
```
● cloudflared.service - cloudflared
   Loaded: loaded (/etc/systemd/system/cloudflared.service; enabled)
   Active: active (running) since ...
```

---

## Étape 7: Vérification et Tests

### 7.1 Vérifier l'État du Tunnel

```bash
# Obtenir les infos du tunnel
cloudflared tunnel info pensine
```

**Output attendu:**
```
NAME: pensine
ID: <TUNNEL-ID>
CREATED: 2024-01-19T...
CONNECTIONS: 4
```

### 7.2 Lister Tous les Tunnels

```bash
# Voir tous vos tunnels
cloudflared tunnel list
```

### 7.3 Tester l'Accès Public

#### Test MinIO Storage

```bash
# Depuis votre machine locale
curl -I https://storage.pensine.app/minio/health/live

# Output attendu:
# HTTP/2 200
# server: cloudflare
# ...
```

#### Test depuis Internet

1. Depuis votre téléphone (4G/5G, pas WiFi)
2. Ouvrir le navigateur
3. Aller sur: `https://storage.pensine.app/minio/health/live`
4. Vous devriez voir une réponse (ou page MinIO)

---

## Étape 8: Sécuriser les Services (Recommandé)

### 8.1 Activer l'Access Control Cloudflare (Optionnel)

Pour protéger vos endpoints avec authentification:

1. Aller sur: https://dash.cloudflare.com
2. Sélectionner votre domaine
3. **Zero Trust → Access → Applications**
4. Créer une application pour `api.pensine.app`
5. Configurer les règles d'accès (email, IP, etc.)

### 8.2 Configurer les WAF Rules (Optionnel)

1. Dans Cloudflare Dashboard → **Security → WAF**
2. Créer des règles pour bloquer les IPs suspectes
3. Activer le **Bot Fight Mode**

---

## ✅ Checklist de Validation

Avant de marquer cette étape comme terminée:

- [ ] Cloudflared installé et vérifié
- [ ] Tunnel créé avec succès
- [ ] Fichier `~/.cloudflared/config.yml` configuré
- [ ] DNS configurés pour `api.pensine.app` et `storage.pensine.app`
- [ ] Tunnel démarré (foreground ou service)
- [ ] `https://storage.pensine.app/minio/health/live` accessible depuis Internet
- [ ] Tunnel configuré pour démarrer automatiquement (service)

---

## 🔍 Troubleshooting

### Erreur: "Failed to authenticate"

**Cause:** Le certificat n'a pas été créé lors de `cloudflared tunnel login`.

**Solution:**
```bash
# Supprimer l'ancien certificat
rm ~/.cloudflared/cert.pem

# Réauthentifier
cloudflared tunnel login
```

### Erreur: "Tunnel credentials not found"

**Cause:** Le fichier de credentials JSON n'existe pas ou le chemin est incorrect.

**Solution:**
```bash
# Lister les tunnels
cloudflared tunnel list

# Vérifier que le fichier existe
ls ~/.cloudflared/

# Mettre à jour config.yml avec le bon chemin
```

### Erreur: "Connection refused" lors du test

**Cause:** Le service backend (MinIO ou NestJS) n'est pas démarré.

**Solution:**
```bash
# Vérifier que les services Docker tournent
docker-compose ps

# Démarrer si nécessaire
docker-compose up -d

# Vérifier que MinIO répond localement
curl http://localhost:9000/minio/health/live
```

### DNS ne se résout pas

**Cause:** Propagation DNS pas encore terminée.

**Solution:**
```bash
# Vérifier la résolution DNS
dig api.pensine.app

# Attendre quelques minutes et réessayer
# Vider le cache DNS local (macOS):
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

### Tunnel se déconnecte souvent

**Cause:** Connexion Internet instable ou firewall bloquant.

**Solution:**
1. Vérifier votre connexion Internet
2. Vérifier que les ports 7844 et 443 ne sont pas bloqués
3. Essayer avec un VPN différent si applicable

---

## 🛠️ Commandes Utiles

```bash
# Voir les logs du tunnel (foreground)
cloudflared tunnel run pensine

# Voir les logs du service (systemd)
sudo journalctl -u cloudflared -f

# Arrêter le tunnel (foreground)
Ctrl+C

# Arrêter le service (systemd)
sudo systemctl stop cloudflared

# Redémarrer le service (systemd)
sudo systemctl restart cloudflared

# Désinstaller le service
sudo cloudflared service uninstall

# Supprimer un tunnel
cloudflared tunnel delete pensine

# Lister toutes les routes
cloudflared tunnel route dns
```

---

## 📚 Références

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Cloudflare Tunnel GitHub](https://github.com/cloudflare/cloudflared)
- [Cloudflare Zero Trust](https://developers.cloudflare.com/cloudflare-one/)

---

**Prochaine étape:** Initialiser le projet mobile (Expo + React Native)
