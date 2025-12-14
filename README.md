# Auth API (PHP)

API d’authentification “plug & play” en PHP, basée sur des *actions* (query/body) et pensée pour une intégration simple côté front.

## Fonctionnalités

- Inscription + vérification e-mail (avec lien d’annulation)
- Connexion / déconnexion / déconnexion de tous les appareils (sessions en DB)
- “Me” (récupération du profil de l’utilisateur connecté)
- Reset mot de passe par e-mail
- Changement d’e-mail avec double notification (ancien + nouvelle adresse)
- Profil: update du username
- Avatar: upload (JSON base64 ou multipart) + récupération via endpoint
- Suppression de compte (confirmation par mot de passe)

## Stack

- PHP (entrée `index.php`)
- `vlucas/phpdotenv` (variables d’environnement)
- `catfan/medoo` (DB)
- `phpmailer/phpmailer` (SMTP)

## Pré-requis

- PHP 8+ recommandé
- Extensions PHP: `pdo` (selon DB), `openssl`, `mbstring`, `fileinfo`, `gd` (avatar)
- Une base de données avec les tables nécessaires (`users`, `sessions`, `email_verification_tokens`, `password_reset_tokens`)

## Configuration

Le projet charge automatiquement un fichier `.env` à la racine.

Variables attendues (minimum) :

```env
# App
BASE_API=https://example.com/auth/
SESSION_DURATION_HOURS=168
ALLOWED_MIME=image/png,image/jpeg,image/webp

# Database
DB_TYPE=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=app
DB_PASS=secret
DB_NAME=auth

# SMTP
SMTP_HOST=smtp.example.com
SMTP_AUTH=true
SMTP_USER=user
SMTP_PASS=pass
SMTP_PORT=465
SMTP_FROM=no-reply@example.com
SMTP_NAME=Auth
```

Notes importantes :

- Les cookies de session sont configurés avec le domaine `.shinederu.lol` dans le code (`login`, `logout`, `deleteAccount`). En auto-hébergement, adaptez ce domaine à votre propre environnement.
- Les CORS sont commentés dans `index.php` (Nginx gère les CORS). Si besoin, activez `CorsMiddleware::apply()`.

## Lancer en local (dev)

Le dépôt inclut déjà `vendor/` et `vendor/autoload.php`.

```bash
php -S 127.0.0.1:8000 index.php
```

Ensuite: `http://127.0.0.1:8000/?action=...`

## Authentification (session)

Après `login`, un cookie `sid` (HTTP-only, Secure) est envoyé.

Pour les endpoints protégés, vous pouvez aussi passer la session via header :

- `X-Session-Id: <sid>` (recommandé)
- `Session-Id: <sid>` (legacy)

## Format des réponses

Succès :

```json
{ "success": true, "message": "...", "data": { } }
```

Erreur :

```json
{ "success": false, "error": "..." }
```

## Endpoints

Le routing est basé sur la méthode HTTP + `action`.

| Méthode | action | Auth | Description |
|---|---|---:|---|
| GET | `me` | ✅ | Retourne l’utilisateur connecté |
| GET | `getAvatar` | ❌ | Retourne l’avatar PNG (`user_id`) |
| POST | `register` | ❌ | Inscription |
| POST | `verifyEmail` | ❌ | Vérifie un token e-mail |
| POST | `revokeRegister` | ❌ | Annule une inscription (token) |
| POST | `login` | ❌ | Connexion (cookie `sid`) |
| POST | `logout` | ✅ | Déconnexion (session courante) |
| POST | `logoutAll` | ✅ | Déconnexion de toutes les sessions |
| POST | `requestPasswordReset` | ❌ | Envoie un e-mail de reset |
| PUT | `resetPassword` | ❌ | Change le mot de passe via token |
| PUT | `requestEmailUpdate` | ✅ | Demande changement d’e-mail (double mail) |
| POST | `confirmEmailUpdate` | ❌ | Confirme le changement d’e-mail (token) |
| POST | `revokeEmailUpdate` | ❌ | Annule le changement d’e-mail (token) |
| POST | `updateProfile` | ✅ | Met à jour le username |
| POST | `updateAvatar` | ✅ | Met à jour l’avatar (base64 ou multipart) |
| DELETE | `deleteAccount` | ✅ | Supprime le compte (password) |

## Exemples (cURL)

### Inscription

```bash
curl -X POST "http://127.0.0.1:8000/" \
  -H "Content-Type: application/json" \
  -d '{"action":"register","username":"demo","email":"demo@example.com","password":"Password123!","password_confirm":"Password123!"}'
```

### Connexion

```bash
curl -i -X POST "http://127.0.0.1:8000/" \
  -H "Content-Type: application/json" \
  -d '{"action":"login","username":"demo","password":"Password123!"}'
```

### Profil courant (`me`) via header

```bash
curl "http://127.0.0.1:8000/?action=me" \
  -H "X-Session-Id: <sid>"
```

### Mise à jour avatar (multipart)

```bash
curl -X POST "http://127.0.0.1:8000/" \
  -H "X-Session-Id: <sid>" \
  -F "action=updateAvatar" \
  -F "file=@avatar.png"
```

### Récupérer un avatar

```bash
curl -L "http://127.0.0.1:8000/?action=getAvatar&user_id=123" --output avatar.png
```

## Schéma DB (indicatif)

Le projet suppose au minimum les tables suivantes (adaptez types/indices à votre SGBD) :

- `users`: `id`, `username`, `email`, `password_hash`, `role`, `email_verified`, `avatar_url`, `avatar_image`, `created_at`
- `sessions`: `id`, `user_id`, `expires_at`
- `email_verification_tokens`: `user_id`, `token`, `expires_at`, `new_email` (nullable)
- `password_reset_tokens`: `user_id`, `token`, `expires_at`
