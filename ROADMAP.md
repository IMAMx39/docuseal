# SignFlow — Roadmap de Développement

> Plateforme open-source de signature électronique. Alternative à DocuSign/HelloSign.
> **Stack : Next.js 14 · Node.js (NestJS) · PostgreSQL · Tailwind CSS**

---

## Vue d'ensemble

```
MVP  (Phases 1–3) ───── ~13 semaines  → Produit fonctionnel de bout en bout
V1.0 (Phases 4–5) ──── ~6 semaines   → Signature numérique légale + API REST
V2.0 (Phases 6–7) ──── ~10 semaines  → Dashboard avancé + intégrations
PROD (Phase 8)    ──────~4 semaines   → Scalabilité + conformité RGPD
```

---

## Architecture cible

```
signflow/
├── apps/
│   ├── web/          # Next.js 14 (App Router) — Frontend & API Routes
│   └── api/          # NestJS — Backend métier (optionnel si API Routes suffisent)
├── packages/
│   ├── ui/           # Composants partagés (shadcn/ui + Tailwind)
│   ├── pdf/          # Librairie PDF (pdf-lib, PDF.js)
│   └── shared/       # Types TypeScript partagés
├── docker-compose.yml
└── .env.example
```

**Technologies principales :**

| Couche | Technologie | Rôle |
|---|---|---|
| Frontend | Next.js 14 (App Router) | UI + SSR |
| Backend | NestJS / Next.js API Routes | API REST |
| Base de données | PostgreSQL | Données métier |
| ORM | Prisma | Schéma & requêtes |
| Workers | BullMQ + Redis | Jobs asynchrones |
| PDF | pdf-lib + PDF.js | Génération & rendu |
| Stockage | S3 / MinIO (self-hosted) | Fichiers & blobs |
| Auth | NextAuth v5 | Sessions & tokens |
| Email | Nodemailer / Resend | Envoi d'emails |
| UI | Tailwind CSS + shadcn/ui | Design system |
| Tests | Jest + Playwright | Unit & E2E |
| CI/CD | GitHub Actions | Automatisation |

---

## PHASE 1 — Fondations (Semaines 1–4)

### 1.1 Infrastructure & Setup

- [ ] Initialiser le monorepo (Turborepo)
- [ ] Configurer ESLint, Prettier, TypeScript strict
- [ ] Docker Compose : PostgreSQL + Redis + MinIO + App
- [ ] Pipeline CI/CD GitHub Actions (lint, tests, build)
- [ ] Variables d'environnement (`.env.example` documenté)
- [ ] Logging structuré (Pino)

### 1.2 Authentification

- [ ] Inscription / Connexion (email + mot de passe)
- [ ] NextAuth v5 avec JWT + refresh tokens
- [ ] 2FA via TOTP (QR code — bibliothèque `otpauth`)
- [ ] Réinitialisation mot de passe (lien par email)
- [ ] Middleware de protection des routes

### 1.3 Multi-tenancy & Organisations

- [ ] Modèle `Account` (workspace isolé)
- [ ] Invitation de membres par email
- [ ] Rôles : `owner`, `admin`, `member`, `viewer`
- [ ] Clés API : test vs production (séparées)
- [ ] Onboarding (wizard de création de compte)

### 1.4 Schéma base de données (Prisma)

```prisma
model User {
  id            String   @id @default(cuid())
  email         String   @unique
  passwordHash  String
  totpSecret    String?
  accounts      AccountUser[]
  createdAt     DateTime @default(now())
}

model Account {
  id          String   @id @default(cuid())
  name        String
  uuid        String   @unique @default(uuid())
  plan        String   @default("free")
  settings    Json     @default("{}")
  users       AccountUser[]
  templates   Template[]
  submissions Submission[]
  apiTokens   ApiToken[]
  createdAt   DateTime @default(now())
}

model AccountUser {
  userId    String
  accountId String
  role      String   @default("member")
  user      User     @relation(fields: [userId], references: [id])
  account   Account  @relation(fields: [accountId], references: [id])
  @@id([userId, accountId])
}

model ApiToken {
  id        String   @id @default(cuid())
  accountId String
  tokenHash String   @unique
  mode      String   @default("production") // "test" | "production"
  name      String?
  account   Account  @relation(fields: [accountId], references: [id])
  createdAt DateTime @default(now())
}
```

**Livrable :** App avec auth fonctionnelle, dashboard vide, gestion d'équipe.

---

## PHASE 2 — Gestion des Templates (Semaines 5–8)

### 2.1 Upload & Traitement des Documents

- [ ] Upload PDF direct vers S3/MinIO (presigned URL)
- [ ] Détection des champs AcroForm existants (`pdf-lib`)
- [ ] Génération des images de prévisualisation (1 image/page via `pdfjs-dist`)
- [ ] Conversion DOCX → PDF (service Gotenberg en Docker)
- [ ] Conversion Image (PNG/JPG) → PDF
- [ ] Stockage des métadonnées (nb pages, dimensions, orientation)

### 2.2 Template Builder — Éditeur Visuel

> C'est la fonctionnalité la plus complexe (3–4 semaines dédiées).

- [ ] Affichage PDF page par page (PDF.js canvas)
- [ ] Drag-and-drop de champs sur le PDF (`@dnd-kit/core`)
- [ ] Redimensionnement des champs (resize handles)
- [ ] Zoom et navigation entre pages
- [ ] **8 types de champs core :**
  - `signature` — Pad de signature canvas
  - `initials` — Initiales
  - `text` — Champ texte libre
  - `date` — Sélecteur de date
  - `checkbox` — Case à cocher
  - `select` — Liste déroulante
  - `email` — Champ email validé
  - `file` — Upload de fichier
- [ ] Attribution des champs aux signataires (couleurs distinctes)
- [ ] Champ requis / optionnel
- [ ] Sauvegarde schema JSON (position x/y, taille, type, signataire)
- [ ] Undo/Redo (historique des actions)

### 2.3 Gestion des Templates

- [ ] CRUD templates (créer, éditer, dupliquer, archiver, supprimer)
- [ ] Dossiers / catégories
- [ ] Prévisualisation thumbnail (1ère page)
- [ ] Partage de template entre membres du compte
- [ ] Recherche full-text (PostgreSQL `tsvector`)

### 2.4 Schéma base de données

```prisma
model Template {
  id          String   @id @default(cuid())
  accountId   String
  name        String
  folderId    String?
  schema      Json     // structure des documents
  fields      Json     // définitions des champs
  submitters  Json     // rôles des signataires
  account     Account  @relation(fields: [accountId], references: [id])
  submissions Submission[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model TemplateFolder {
  id        String   @id @default(cuid())
  accountId String
  name      String
  parentId  String?
  createdAt DateTime @default(now())
}
```

**Livrable :** Upload PDF + placement de champs + sauvegarde template.

---

## PHASE 3 — Workflow de Signature (Semaines 9–13)

### 3.1 Création & Envoi des Submissions

- [ ] Créer une submission depuis un template
- [ ] Assigner les signataires (email + nom + ordre)
- [ ] Mode **séquentiel** (A signe → puis B) ou **parallèle** (tous en même temps)
- [ ] Pré-remplissage de variables dynamiques (`{{nom}}`, `{{date}}`)
- [ ] Date d'expiration configurable
- [ ] Envoi email d'invitation (lien unique sécurisé avec token)
- [ ] Relance manuelle des signataires en attente

### 3.2 Interface de Signature (Page Publique)

> Accessible sans login — optimisée mobile.

- [ ] Rendu PDF + champs superposés (positions exactes)
- [ ] Navigation guidée entre champs obligatoires
- [ ] **Modes de signature :**
  - Dessin canvas (Signature Pad library)
  - Texte en police calligraphique
  - Upload d'image
- [ ] Vérification identité :
  - Code OTP par email (avant signature)
  - Code OTP par SMS (optionnel — Twilio)
- [ ] Remplissage de tous les types de champs
- [ ] Bouton "Refuser" avec motif
- [ ] Confirmation et récapitulatif avant soumission
- [ ] Page de succès avec téléchargement

### 3.3 Génération du PDF Final

- [ ] Remplir les champs PDF avec les valeurs (`pdf-lib`)
- [ ] Intégrer l'image de signature à la position exacte
- [ ] Aplatir le PDF (flatten — rend non éditable)
- [ ] Générer le **PDF d'audit trail** :
  - Historique des événements
  - IP + user-agent + timestamps
  - Hash du document
- [ ] Assembler le PDF final (document + audit trail)
- [ ] Notifier toutes les parties (email + PDF joint)
- [ ] Stocker le PDF final en S3/MinIO

### 3.4 Schéma base de données

```prisma
model Submission {
  id          String   @id @default(cuid())
  accountId   String
  templateId  String
  status      String   @default("pending") // pending|completed|expired|declined
  createdBy   String
  expiresAt   DateTime?
  account     Account    @relation(fields: [accountId], references: [id])
  template    Template   @relation(fields: [templateId], references: [id])
  submitters  Submitter[]
  events      SubmissionEvent[]
  createdAt   DateTime @default(now())
}

model Submitter {
  id          String   @id @default(cuid())
  submissionId String
  email       String
  name        String?
  role        String
  order       Int      @default(0)
  status      String   @default("pending") // pending|opened|completed|declined
  tokenHash   String   @unique
  values      Json     @default("{}")     // réponses aux champs
  ip          String?
  userAgent   String?
  completedAt DateTime?
  submission  Submission @relation(fields: [submissionId], references: [id])
  events      SubmissionEvent[]
  createdAt   DateTime @default(now())
}

model SubmissionEvent {
  id           String   @id @default(cuid())
  submissionId String
  submitterId  String?
  eventType    String   // viewed|started|completed|declined|bounced...
  data         Json     @default("{}")
  ip           String?
  createdAt    DateTime @default(now())
}
```

**Livrable :** Envoi → Signature → PDF généré → Notifications. Flux complet fonctionnel.

---

## PHASE 4 — Signature Numérique Légale (Semaines 14–16)

> Partie cryptographique — valeur légale du document.

### 4.1 Infrastructure PKI (Public Key Infrastructure)

- [ ] Génération certificat CA Root (OpenSSL / `node-forge`)
- [ ] Génération Sub-CA → Leaf certificate par soumission
- [ ] Support certificats PKCS#12 custom (upload admin)
- [ ] Stockage sécurisé des clés privées (chiffré en base)

### 4.2 Signature PKCS#7 / CMS

- [ ] Calcul du byte range pour intégration dans le PDF
- [ ] Génération signature RSA-2048 + SHA-256
- [ ] Embedding de la signature dans le PDF (format standard Adobe)
- [ ] Intégration horodatage TSA (Time Stamping Authority)
- [ ] LTV (Long-Term Validation) — embed CRL/OCSP responses
- [ ] Outil de vérification de signature (`/api/tools/verify`)

### 4.3 Conformité légale

- [ ] Conformité **eIDAS** (UE) — niveau SES (Simple Electronic Signature)
- [ ] Conformité **ESIGN Act** (USA)
- [ ] Audit trail immutable et signé cryptographiquement
- [ ] Conservation des preuves 10 ans minimum

**Livrable :** PDF avec signature numérique vérifiable par Adobe Acrobat.

---

## PHASE 5 — API REST & Webhooks (Semaines 17–19)

### 5.1 API REST Complète

```
POST   /api/v1/templates                    Créer template
GET    /api/v1/templates                    Lister templates
GET    /api/v1/templates/:id                Détails template
PATCH  /api/v1/templates/:id                Modifier template
DELETE /api/v1/templates/:id                Supprimer template

POST   /api/v1/submissions                  Créer submission
GET    /api/v1/submissions                  Lister submissions
GET    /api/v1/submissions/:id              Détails submission
GET    /api/v1/submissions/:id/documents    Télécharger PDFs

GET    /api/v1/submitters/:id               Info signataire
PATCH  /api/v1/submitters/:id               Mettre à jour signataire

POST   /api/v1/attachments                  Upload fichier
POST   /api/v1/tools/merge                  Fusionner PDFs
POST   /api/v1/tools/verify                 Vérifier signature
```

- [ ] Authentification `X-Auth-Token` (hash SHA-256 du token)
- [ ] Rate limiting : 60 req/min (test), 600 req/min (prod)
- [ ] Pagination cursor-based (`after`, `before`, `limit`)
- [ ] Gestion d'erreurs standardisée (RFC 7807)
- [ ] Documentation OpenAPI/Swagger auto-générée

### 5.2 Système de Webhooks

- [ ] **Événements :**
  - `submission.created` — Nouvelle submission
  - `form.viewed` — Signataire a ouvert le formulaire
  - `form.started` — Signataire a commencé à remplir
  - `form.completed` — Signataire a complété
  - `form.declined` — Signataire a refusé
  - `submission.completed` — Tous les signataires ont signé
  - `submission.expired` — Submission expirée
- [ ] Retry logic : 3 tentatives avec backoff exponentiel (5s, 30s, 5min)
- [ ] Signature HMAC-SHA256 du payload (`X-Signature` header)
- [ ] Interface de gestion (URL, événements souscrits, logs)
- [ ] Rejouer un webhook manuellement

**Livrable :** API documentée + webhooks fonctionnels. Intégrable avec Zapier/Make.

---

## PHASE 6 — Dashboard & Analytics (Semaines 20–22)

### 6.1 Dashboard Principal

- [ ] Vue d'ensemble : soumissions en attente / complétées / expirées
- [ ] Tableau des submissions avec filtres (statut, date, template)
- [ ] Recherche full-text (nom, email, valeurs des champs)
- [ ] Timeline des événements par submission
- [ ] Téléchargement des documents complétés
- [ ] Relance en 1 clic des signataires en attente
- [ ] Annulation / archivage d'une submission

### 6.2 Analytics

- [ ] Taux de complétion (par template, par période)
- [ ] Temps moyen de signature
- [ ] Graphiques (soumissions par jour/mois)
- [ ] Export CSV/XLSX des données

### 6.3 Paramètres du Compte

- [ ] Branding (logo, couleurs, nom affiché)
- [ ] SMTP custom (configurer son propre serveur email)
- [ ] Templates d'emails personnalisés (invitation, rappel, confirmation)
- [ ] Gestion des membres (inviter, supprimer, changer de rôle)
- [ ] Clés API (générer, révoquer, voir les logs d'utilisation)
- [ ] Facturation et plans (intégration Stripe)

### 6.4 Bulk Send

- [ ] Import CSV/XLSX pour envoi massif
- [ ] Correspondance colonnes → variables du template
- [ ] Preview avant envoi (vérification des données)
- [ ] Tracking progression batch (X/Y complétés)

**Livrable :** Interface complète et professionnelle. Prêt pour les utilisateurs beta.

---

## PHASE 7 — Fonctionnalités Avancées (Semaines 23–28)

### 7.1 Champs Conditionnels & Formules

- [ ] Règles de visibilité : `si champ_A == "oui" → afficher champ_B`
- [ ] Éditeur visuel de conditions (no-code)
- [ ] Calculs : sommes, concatenation de champs
- [ ] Champs pré-remplis depuis des variables externes

### 7.2 Types de Champs Avancés

- [ ] `phone` — Numéro de téléphone avec indicatif pays
- [ ] `number` — Champ numérique avec validation
- [ ] `radio` — Boutons radio (choix exclusif)
- [ ] `payment` — Intégration Stripe (paiement lors de la signature)
- [ ] `image` — Upload d'une image dans le PDF
- [ ] `stamp` — Tampon/cachet d'entreprise

### 7.3 Intégrations

- [ ] **Zapier** — Triggers & Actions
- [ ] **Make (Integromat)** — Module dédié
- [ ] **Slack** — Notifications de complétion
- [ ] **Google Drive** — Export automatique
- [ ] **Notion** — Sync des données

### 7.4 SDK d'Intégration (Embed)

- [ ] iFrame embed (signer depuis une autre app)
- [ ] SDK JavaScript npm (`@signflow/embed`)
- [ ] Callbacks : `onComplete`, `onDecline`, `onError`
- [ ] Redirect URL configurable après signature
- [ ] Personnalisation CSS de l'interface embarquée

### 7.5 Multi-langue

- [ ] Français, Anglais, Espagnol, Portugais, Allemand, Arabe
- [ ] Support RTL (arabe, hébreu)
- [ ] Langue configurable par signataire

**Livrable :** Plateforme feature-complete, intégrable, multi-langue.

---

## PHASE 8 — Production & Scale (Semaines 29–32)

### 8.1 Performance & Scalabilité

- [ ] CDN pour assets et prévisualisations (CloudFront / Cloudflare)
- [ ] Cache Redis pour templates et sessions
- [ ] Auto-scaling workers BullMQ
- [ ] Optimisation requêtes PostgreSQL (index, query plans)
- [ ] Tests de charge (k6) — objectif : 1000 soumissions simultanées

### 8.2 Monitoring & Observabilité

- [ ] Error tracking (Sentry)
- [ ] APM (Datadog ou Grafana + Tempo)
- [ ] Logs structurés centralisés (Loki ou ELK)
- [ ] Alertes automatiques (downtime, erreurs, quota)
- [ ] Dashboard de santé (uptime, latence, files workers)

### 8.3 Sécurité & Conformité

- [ ] Audit de sécurité (OWASP Top 10)
- [ ] Chiffrement des données sensibles en base (certificats, clés)
- [ ] Politique de rétention des données configurable
- [ ] **RGPD :**
  - Droit à l'effacement (suppression compte + données)
  - Export des données personnelles
  - Consentement tracé
  - DPA (Data Processing Agreement) template
- [ ] Pen test externe (avant launch public)

### 8.4 Déploiement

- [ ] Dockerfile multi-stage optimisé
- [ ] Docker Compose (self-hosted, 1 commande)
- [ ] Helm Chart (Kubernetes)
- [ ] One-click deploy : Railway, Render, DigitalOcean
- [ ] Backup automatique PostgreSQL (daily → S3)
- [ ] Disaster recovery documenté

**Livrable :** Plateforme production-ready. Launch public possible.

---

## Jalons et livrables

| Milestone | Phase | Livrable |
|---|---|---|
| **M1 — Auth** | Fin Phase 1 | Authentification + Multi-tenant |
| **M2 — Builder** | Fin Phase 2 | Upload PDF + éditeur de champs |
| **M3 — MVP** | Fin Phase 3 | Envoi → Signature → PDF généré |
| **M4 — Legal** | Fin Phase 4 | Signature numérique PKCS#7 |
| **M5 — API** | Fin Phase 5 | API REST + Webhooks documentés |
| **M6 — Beta** | Fin Phase 6 | Dashboard complet + Bulk send |
| **M7 — V2** | Fin Phase 7 | Champs avancés + Intégrations + SDK |
| **M8 — PROD** | Fin Phase 8 | Scalabilité + RGPD + Launch |

---

## Structure de fichiers recommandée (Next.js)

```
signflow/
├── app/                          # Next.js App Router
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── (dashboard)/
│   │   ├── templates/
│   │   │   ├── page.tsx          # Liste templates
│   │   │   ├── new/page.tsx      # Upload + créer
│   │   │   └── [id]/
│   │   │       ├── page.tsx      # Détails
│   │   │       └── edit/page.tsx # Builder éditeur
│   │   ├── submissions/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   └── settings/
│   │       ├── page.tsx
│   │       └── api-keys/page.tsx
│   ├── sign/
│   │   └── [token]/page.tsx      # Page de signature publique
│   └── api/
│       ├── auth/[...nextauth]/
│       └── v1/
│           ├── templates/
│           ├── submissions/
│           ├── submitters/
│           └── webhooks/
├── components/
│   ├── template-builder/         # Éditeur PDF
│   ├── signature-pad/            # Canvas signature
│   ├── pdf-viewer/               # Rendu PDF.js
│   └── ui/                       # shadcn/ui components
├── lib/
│   ├── pdf/                      # Génération & manipulation PDF
│   ├── auth/                     # NextAuth config
│   ├── storage/                  # S3/MinIO adapter
│   ├── email/                    # Nodemailer/Resend
│   └── crypto/                   # Certificats & signatures
├── prisma/
│   └── schema.prisma
├── workers/
│   └── jobs/                     # BullMQ jobs
└── docker-compose.yml
```

---

## Points critiques à ne pas sous-estimer

1. **Template Builder** — Synchroniser la position des champs entre l'éditeur et le PDF réel requiert une gestion précise des coordonnées (PDF ≠ pixels écran).
2. **Signature PKCS#7** — Utiliser `node-forge` ou `@signpdf/signpdf`. Le byte range est délicat à calculer.
3. **Génération PDF** — Tester avec des PDFs complexes (polices embarquées, rotation de pages, formulaires XFA).
4. **Conversion DOCX→PDF** — Gotenberg (service Docker) est la solution la plus fiable.
5. **Performance mobile** — La page de signature doit charger en < 3s sur 3G.
6. **Conformité légale** — Consulter un juriste pour les marchés cibles (EIDAS EU vs ESIGN USA).

---

*SignFlow — Construit avec ❤️ comme alternative open-source à DocuSign.*
