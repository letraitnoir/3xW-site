# FICHE PROJET — « LE QG » 🖤

> **Plateforme unifiée de gestion du salon Le Trait Noir**
> Document de référence destiné à un **agent IA développeur senior**. Il est autoportant : toutes les informations nécessaires pour lancer la construction sont ici. Lis-le intégralement avant d'écrire la première ligne de code.

| Méta | Valeur |
|---|---|
| Nom du produit | **Le QG** |
| URL de production | `https://qg.letraitnoir.fr` |
| Commanditaire / utilisateur unique | Ben — gérant du salon de tatouage & piercing **Le Trait Noir** (France) |
| Contact | bletraitnoir@gmail.com |
| Repo de build (à créer) | `letraitnoir/le-qg` (ce document vit dans `letraitnoir/3xW-site`) |
| Statut | Fiche validée par Ben — prêt pour Phase 0 |
| Date | 2026-07-20 |

---

## 1. Vision & contexte métier

### 1.1 Qui est Ben

Ben est tatoueur et perceur, seul aux commandes du salon **Le Trait Noir**. Il gère aussi un studio web (« 3xW ») : il est à l'aise avec les outils numériques, utilise déjà Claude au quotidien, mais **il ne veut plus s'adapter aux outils — il veut un outil qui s'adapte à lui**.

### 1.2 Le process actuel (à remplacer / automatiser)

| Outil actuel | Usage | Devenir |
|---|---|---|
| **Notion** (payant) | CRM clients (« liste crm »), fiches projets partagées avec les clients, traçabilité encres/aiguilles, factures fournisseurs, avis Google, FAQ, flashs | **Remplacé à 100 % par Le QG → résiliation** |
| **Spark** (payant) | Client mail (adresses Gmail synchronisées) | Remplacé par la « Boîte triée » du QG + Gmail web |
| **Plaud** (dictaphone IA) | Enregistrement des entretiens au salon ; le résumé arrive par mail | Conservé — Le QG **ingère automatiquement** ces mails |
| **Planity** | Prise de rendez-vous (⚠️ **pas d'API**) | **Conservé obligatoirement** — synchronisé indirectement (voir §6.5) |
| **Google Calendar** | Miroir de Planity, alimenté par une routine Claude+Chrome | Conservé — **source de vérité des RDV** lue par Le QG |
| **Routine Claude « tri des mails du matin »** | Compare les mails entrants au CRM Notion : client existant → enrichit la fiche ; inconnu → crée la fiche + projet | **Remplacée par la routine `mail-triage`** (§6.4) qui écrit dans la base du QG (forfait Claude Max, zéro frais API) |
| **Routine Claude « Planity → GCal »** | Navigue sur Planity via Chrome et recopie les RDV dans Google Calendar | **Conservée telle quelle** (seul moyen sans API), surveillée par Le QG |

### 1.3 La semaine type de Ben (le workflow à servir)

- **Lundi = jour dessin.** Ben ouvre Planity, regarde les RDV de la semaine (mardi → samedi), puis va dans Notion chercher les infos de chaque projet (zone, références, échanges). C'est long, plein de clics, réparti sur 3 outils. → **Le QG doit générer automatiquement un « Brief du lundi »** : chaque RDV de la semaine avec fiche client, projet, zone, références visuelles, extraits des derniers mails, transcript d'entretien s'il existe, prix estimé.
- **Au salon**, les entretiens de prise de RDV sont enregistrés (Plaud) ; le résumé arrive par mail → doit finir automatiquement dans la fiche client.
- **Chaque matin**, les mails entrants doivent être triés et rapprochés des clients automatiquement.

### 1.4 Objectifs produit (par ordre de priorité)

1. **Réduire le nombre de clics** : toute information à ≤ 2 clics ; zéro ressaisie.
2. **Automatiser un maximum** : tri des mails, enrichissement des fiches, brief hebdo, notifications clients, relance J+3 — avec **validation humaine** pour tout ce qui sort vers un client ou modifie une fiche de façon non triviale.
3. **Résilier Notion** (et Spark) : le QG couvre tout le périmètre du QG Notion actuel.
4. **Design niveau AWWWards** : direction artistique « Encre & papier » (voir §7). L'outil doit être *beau*, agréable à parcourir, et irréprochable sur mobile comme sur desktop.

---

## 2. Décisions produit verrouillées (validées par Ben)

Ces décisions ont été prises explicitement avec Ben. **Ne pas les remettre en question sans lui demander.**

1. **Portail client inclus dès le départ** : chaque client a un **lien privé** (sans compte) vers son espace projet — croquis, commentaires, validation, fiches de soins. Il remplace les pages Notion partagées actuelles.
2. **Mono-utilisateur** : Ben uniquement côté interne. Auth simple et sécurisée (email + mot de passe + TOTP). Pas de gestion de rôles.
3. **Automatisations : « IA en routines Claude Max, mécanique dans l'app »** — objectif : **zéro frais API variable**, toute l'intelligence tourne sur le forfait Claude Max de Ben :
   - **Routines Claude Code** (déclencheurs cron — **intervalle minimum 1 h**) : tri + extraction des mails (horaire), ingestion des transcripts Plaud (horaire), brief du lundi (lundi 06h00), rédaction des mails J+3 (quotidien), extraction des factures PDF (horaire), tattooversaire (hebdo), pont Planity → Google Calendar (existant) et missions ad hoc.
   - **App (0 token Claude)** : recopie brute Gmail + Google Agenda (crons 15 min, API Google directes), envoi des mails à partir de modèles validés, notifications push, portail, file de validation, planification du suivi cicatrisation, backups, heartbeat Planity.
   - **Pattern « work-queue »** : une routine ne peut pas être déclenchée par un événement de l'app ; l'app dépose le travail dans des tables à statut « à traiter », la routine horaire ramasse tout ce qui attend (via MCP Supabase) et écrit ses propositions dans `review_queue`. Latence acceptée : ≤ 1 h pour l'intelligence, ≤ 15 min pour la mécanique.
4. **Notifications & validations pour Ben : dans l'app + push mobile.** Le QG est une **PWA installable** ; les demandes de validation arrivent en push (web push), approbation en un tap. Pas de Telegram/WhatsApp. Fallback : digest mail quotidien à 07h30.
5. **Migration totale de Notion** : données + contenus des pages + images/croquis. Étape par étape, avec passes manuelles sur les cas non parsés.
6. **Périmètre = tout le QG Notion** : clientèle, projets, traçabilité, factures fournisseurs, FAQ, flashs. **Exception : le module « Avis Google » est SUPPRIMÉ** (la base Notion correspondante sera juste exportée/archivée, pas migrée). Le mail J+3 contient simplement le lien pour laisser un avis Google.
7. **Pas de client mail complet** : le QG lit, trie et lie les mails, mais **on ne répond pas depuis le QG** — un clic sur l'adresse du client ouvre Gmail (deeplink). Exception : les **mails transactionnels automatisés** (notification croquis, J+3) envoyés par l'app.
8. **Flashs : rapatriement.** Les flashs vivent aujourd'hui sur un **autre compte Supabase** (celui de flash.letraitnoir.fr). On migre tables + assets dans le projet Supabase du QG, puis on rebranche flash.letraitnoir.fr et l'outil d'essayage AR dessus.
9. **Fonctionnalités portail demandées par Ben** :
   - Encart promotionnel **abonnements** (abonnement.letraitnoir.fr) sur chaque espace client.
   - **Flashs aléatoires** (statut Disponible) affichés à chaque visite de l'espace client + lien vers flash.letraitnoir.fr.
   - **Notification automatique au client quand un croquis est publié** sur son espace : mail type « Ça y est, Ben a ajouté un dessin pour ton projet. Tu peux le découvrir ici → [lien privé] ». Débrayable envoi par envoi.
   - **Mail J+3 post-RDV** : « tout va bien après ton passage ? » + lien avis Google. **Toujours validé par Ben avant envoi** (certains clients ne viennent pas / prestation partielle) : proposition en file de validation + push mobile.
10. **Design « Encre & papier »** : noir profond / blanc papier, typographie forte, esprit galerie d'art, micro-animations (§7).
11. **Trois modules ajoutés au périmètre (arbitrage backlog du 2026-07-20)** : **consentement pré-tatouage numérique** signé depuis le portail avant la séance, **suivi cicatrisation** (photo demandée au client à J+15/J+30 via le portail), **tattooversaire** (proposition de mail 1 an après le tatouage). Écartés définitivement : stats (Planity les fournit), relances projets, stock consommables (Planity).

---

## 3. Architecture technique

### 3.1 Stack retenue

| Couche | Choix | Justification |
|---|---|---|
| Framework | **Next.js 15+ (App Router, TypeScript, React Server Components)** | Une seule base de code pour l'app interne, le portail client (SSR + tokens) et les crons (route handlers). Déploiement Vercel natif. Excellent support des ambitions design (`next/font`, `next/image`, streaming). |
| UI | **Tailwind CSS 4 + composants maison** (+ Framer Motion pour le motion) | L'exigence AWWWards impose un design system custom (§7). Possibilité d'utiliser des primitives headless (Radix) pour dialogs/popovers, entièrement re-stylées. **Pas de thème shadcn par défaut.** |
| Base de données | **Supabase Postgres — projet EXISTANT `wwvjxswrilvvpfsiwlvh`** (eu-west-1, Postgres 17), schéma `public`, nouvelles tables | Infra déjà en place (tables `leads`, `audits`, `newsletter` du site 3xW — aucune collision de noms), un seul billing, co-localisation avec les futurs flashs rapatriés. **Passer le projet en plan Supabase Pro** : backups quotidiens + PITR, pas de pause d'inactivité (outil métier quotidien + obligation légale de traçabilité). |
| Auth interne | **Supabase Auth** : email + mot de passe pour Ben, **TOTP/MFA activé**, signups désactivés | Un seul utilisateur. Middleware Next.js protège tout le groupe de routes `/(app)`. |
| Portail client | **Sans compte.** Token opaque 256 bits par client, **stocké hashé (SHA-256)**, révocable, route `/portail/[token]` | Réplique le modèle « lien Notion partagé » que les clients connaissent. Rendu 100 % server-side avec le service role après validation du token — **jamais** d'accès Supabase direct depuis le navigateur du client. |
| Storage | **Supabase Storage, buckets privés** : `croquis`, `realisations`, `tracabilite`, `factures`, `flashs`, `docs`, `backups` | Signed URLs ≤ 1 h pour le portail ; transformations d'images intégrées pour les miniatures ; RLS sur `storage.objects`. |
| Jobs / crons | **Vercel Cron (plan Pro) → route handlers `/api/cron/*`** protégés par `CRON_SECRET`, + tables `job_queue` / `automation_runs` | Les crons de l'app sont **exclusivement mécaniques (0 token Claude)** : recopies, envois de modèles, planification, backups. Chaque tick est borné et idempotent (§6.2). `pg_cron` côté Supabase uniquement pour purges et export logique. |
| IA | **AUCUNE API IA dans l'app.** Toute l'intelligence tourne en **routines Claude Code** sur le forfait Claude Max de Ben (déclencheurs cron, minimum 1 h) | Les routines lisent les tables « à traiter » et écrivent leurs propositions dans `review_queue` via le **MCP Supabase**, et loggent chaque passage dans `automation_runs` (`runner='routine_claude'`). Leurs instructions sont versionnées dans `routines/*.md` du repo (source de vérité unique, rechargée à chaque exécution). Zéro frais API variable. |
| Gmail | **Polling incrémental** : Gmail API `users.history.list` + `historyId` persisté par compte, cron toutes les 15 min | Le push (watch + Pub/Sub GCP) exige trop d'infra pour un gain nul (15 min de latence est indolore ici). Multi-adresses : une ligne `gmail_accounts` par adresse, refresh token OAuth chiffré (Supabase Vault). |
| Google Calendar | **Sync incrémentale par `syncToken`** → table miroir `appointments`, cron 15 min | Planity n'a pas d'API : la routine Claude+Chrome existante alimente GCal ; le QG ne lit QUE GCal. Le miroir en base permet les jointures RDV ↔ client ↔ projet et le brief du lundi. |
| Emails transactionnels | **Gmail API `send`** depuis l'adresse de Ben (recommandé — ton personnel, le client répond à Ben) ; fallback possible : Brevo (déjà utilisé pour la newsletter) | Concerne uniquement : notification croquis publié, mail J+3. Templates versionnés dans le repo, ton de Ben (tutoiement, chaleureux, direct). |
| Push (Ben) | **Web Push (VAPID) via service worker de la PWA** | iOS ≥ 16.4 : nécessite l'installation de la PWA sur l'écran d'accueil (à documenter dans l'onboarding). Fallback : digest mail 07h30. |
| Déploiement | **Vercel, nouveau projet `le-qg`**, domaine `qg.letraitnoir.fr` | Les sous-domaines letraitnoir.fr sont déjà gérés chez Vercel (flash., abonnement.). Vercel Pro requis (crons multiples + exécutions longues). |
| Repo | **Nouveau repo dédié `letraitnoir/le-qg`** | `3xW-site` est un site statique sans build ; mélanger une app Next.js dedans polluerait les deux. CI/CD et cycles de vie indépendants. |

### 3.2 Structure du repo `le-qg`

```
le-qg/
├── src/
│   ├── app/
│   │   ├── (app)/            # App interne (protégée par auth Ben)
│   │   │   ├── page.tsx      # Dashboard « Aujourd'hui / Cette semaine »
│   │   │   ├── clients/  projets/  agenda/  mails/  brief/
│   │   │   ├── tracabilite/  factures/  flashs/  faq/
│   │   │   ├── validations/  reglages/
│   │   ├── (portail)/
│   │   │   └── portail/[token]/   # Espace client (SSR, service role, noindex)
│   │   └── api/
│   │       ├── cron/[job]/route.ts   # Entrée unique des crons (switch par job)
│   │       ├── push/                 # Souscription web push
│   │       └── portail/              # Actions client (commentaire, validation croquis)
│   ├── lib/                  # Clients Supabase/Gmail/GCal, helpers, schémas Zod (review_queue)
│   ├── jobs/                 # Logique de chaque cron mécanique (importée par /api/cron)
│   └── components/           # Design system « Encre & papier »
├── routines/                 # Instructions versionnées des routines Claude Code (§6.4)
│   ├── mail-triage.md  transcript-ingest.md  brief-hebdo.md
│   ├── followup-j3.md  facture-extraction.md  tattooversaire.md
│   └── CONTRAT.md            # Contrat d'interface app ↔ routines (§6.5)
├── supabase/migrations/      # 0001_init.sql, etc.
├── scripts/migration-notion/ # Scripts one-shot idempotents (§8)
└── public/                   # manifest.json PWA, service worker, icônes
```

### 3.3 Variables d'environnement

```
NEXT_PUBLIC_SUPABASE_URL / NEXT_PUBLIC_SUPABASE_ANON_KEY
SUPABASE_SERVICE_ROLE_KEY          # serveur uniquement
GOOGLE_OAUTH_CLIENT_ID / GOOGLE_OAUTH_CLIENT_SECRET
CRON_SECRET                        # vérifié par tous les /api/cron/*
VAPID_PUBLIC_KEY / VAPID_PRIVATE_KEY
PORTAL_TOKEN_PEPPER                # pepper ajouté au hash des tokens portail
```

---

## 4. Modèle de données

Conventions générales : `id uuid primary key default gen_random_uuid()`, `created_at timestamptz default now()`, `updated_at timestamptz` (trigger), soft-delete via `deleted_at timestamptz` sur les entités métier, **RLS activée sur toutes les tables** (policy « authenticated » = Ben ; le portail passe exclusivement par le service role côté serveur). Colonne `notion_page_id text` sur les tables migrées (idempotence de la migration).

### 4.1 Noyau CRM

```sql
-- Clients
create table clients (
  id uuid primary key default gen_random_uuid(),
  nom text not null,                      -- Notion: Nom (title)
  prenom text,                            -- Notion: Prénom
  email citext unique,                    -- Notion: Adresse mail (nullable, unique si présent)
  emails_secondaires text[] default '{}',
  telephone text,                         -- Notion: Téléphone
  adresse_postale text,                   -- Notion: Adresse postale
  date_naissance date,                    -- Notion: Date de naissance (⚠️ mineurs, voir §10)
  newsletter_ok boolean default false,    -- Notion: Newsletter OK
  prestations text[] default '{}',        -- Notion: Prestation réalisée ('tatouage','piercing','tableau')
  notes text,
  source text not null default 'manuel',  -- 'notion' | 'mail_auto' | 'manuel'
  notion_page_id text unique,
  archived_at timestamptz, deleted_at timestamptz,
  created_at timestamptz default now(), updated_at timestamptz default now()
);

-- Tokens du portail client (un client peut avoir plusieurs tokens : rotation/révocation)
create table client_portal_tokens (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id) on delete cascade,
  token_hash text not null unique,        -- sha256(token + PORTAL_TOKEN_PEPPER) ; le clair n'est JAMAIS stocké
  label text, expires_at timestamptz, revoked_at timestamptz,
  last_accessed_at timestamptz, access_count int default 0,
  created_at timestamptz default now()
);

-- Référentiel des zones corporelles (migré depuis le multi-select Notion, ~24 valeurs)
create table body_zones ( id serial primary key, nom text unique not null, ordre int );

-- Projets (un client peut avoir plusieurs projets)
create table projects (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id) on delete cascade,
  titre text not null,
  type text not null default 'tatouage',  -- 'tatouage' | 'piercing' | 'tableau'
  statut text not null default 'demande',
  -- pipeline : demande → consultation → projet_valide → croquis_en_cours → croquis_livre
  --            → croquis_valide → rdv_planifie → realise → retouche → clos
  zones text[] default '{}',              -- valeurs de body_zones
  cover_cicatrice text default 'aucun',   -- 'aucun' | 'cover' | 'cicatrice'
  resume text,                            -- Notion: Résumé du projet
  taille_cm numeric, prix_estime numeric, prix_final numeric,
  devis jsonb,                            -- lignes de devis { libelle, montant }[]
  jalons jsonb default '{}',              -- { projet_valide_le, croquis_livre_le, retouche_prevue_le }
  notion_page_id text unique,
  deleted_at timestamptz,
  created_at timestamptz default now(), updated_at timestamptz default now()
);

-- Rendez-vous : miroir de Google Calendar (lecture seule côté app, seul le sync écrit)
create table appointments (
  id uuid primary key default gen_random_uuid(),
  gcal_event_id text not null unique,
  gcal_calendar_id text not null,
  starts_at timestamptz not null, ends_at timestamptz,
  titre_brut text,                        -- titre de l'événement tel que saisi dans Planity/GCal
  client_id uuid references clients(id) on delete set null,
  project_id uuid references projects(id) on delete set null,
  type_detecte text,                      -- 'tatouage' | 'piercing' | 'conseil' | null
  statut text not null default 'a_venir', -- 'a_venir' | 'realise' | 'annule' | 'no_show'
  followup_j3_statut text default 'na',   -- 'na' | 'propose' | 'envoye' | 'refuse'
  raw jsonb, last_synced_at timestamptz,
  created_at timestamptz default now(), updated_at timestamptz default now()
);

-- Croquis, inspirations, stencils, photos finales
create table drawings (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null references projects(id) on delete cascade,
  type text not null default 'croquis',   -- 'inspiration' | 'croquis' | 'stencil' | 'photo_finale'
  file_id uuid not null references files(id),
  version int default 1,
  statut text default 'propose',          -- 'propose' | 'en_revision' | 'valide_client'
  visible_portail boolean default false,  -- publication → déclenche la notification client (§6.4)
  notification_envoyee_at timestamptz,
  ordre int default 0, deleted_at timestamptz,
  created_at timestamptz default now()
);

create table drawing_comments (
  id uuid primary key default gen_random_uuid(),
  drawing_id uuid not null references drawings(id) on delete cascade,
  author text not null,                   -- 'ben' | 'client'
  body text not null,
  created_at timestamptz default now()
);

-- Fichiers (référence unique vers Supabase Storage)
create table files (
  id uuid primary key default gen_random_uuid(),
  bucket text not null, path text not null,
  mime text, size_bytes bigint, width int, height int,
  sha256 text,                            -- dédoublonnage (migration Notion)
  origin text not null default 'upload',  -- 'upload' | 'notion_migration' | 'email_attachment'
  created_at timestamptz default now(),
  unique (bucket, path)
);
```

### 4.2 Communication & IA

```sql
create table gmail_accounts (
  id uuid primary key default gen_random_uuid(),
  email citext unique not null,
  vault_secret_name text not null,        -- nom du secret Vault contenant le refresh token
  last_history_id text, statut text default 'actif',
  created_at timestamptz default now()
);

create table email_threads (
  id uuid primary key default gen_random_uuid(),
  gmail_account_id uuid not null references gmail_accounts(id),
  gmail_thread_id text not null unique,
  client_id uuid references clients(id) on delete set null,
  project_id uuid references projects(id) on delete set null,
  sujet text, dernier_message_at timestamptz,
  triage text default 'nouveau',          -- 'nouveau'|'client_connu'|'prospect'|'fournisseur'|'planity'|'plaud'|'perso'|'spam'
  triage_by text,                         -- 'ia' | 'ben'
  created_at timestamptz default now(), updated_at timestamptz default now()
);

create table email_messages (
  id uuid primary key default gen_random_uuid(),
  thread_id uuid not null references email_threads(id) on delete cascade,
  gmail_message_id text not null unique,
  from_addr text, to_addrs text[], sent_at timestamptz,
  snippet text, body_text text,           -- texte brut conservé (brief, recherche)
  has_attachments boolean default false,
  direction text not null,                -- 'in' | 'out'
  created_at timestamptz default now()
);

-- Transcripts Plaud (résumés d'entretiens reçus par mail)
create table transcripts (
  id uuid primary key default gen_random_uuid(),
  client_id uuid references clients(id) on delete set null,
  project_id uuid references projects(id) on delete set null,
  source text not null default 'plaud_email',  -- 'plaud_email' | 'upload'
  email_message_id uuid references email_messages(id),
  raw_text text not null,
  resume_ia text,
  extracted jsonb,                        -- { zones, taille, budget, disponibilites, references, ... }
  statut text default 'a_valider',        -- 'a_valider' | 'integre' | 'ignore'
  created_at timestamptz default now()
);

-- Timeline unifiée : colonne vertébrale de la fiche client
create table activities (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id) on delete cascade,
  project_id uuid references projects(id) on delete set null,
  type text not null,  -- 'email_in'|'email_out'|'rdv'|'croquis'|'comment_portail'|'transcript'
                       -- |'note'|'statut_change'|'ia_enrichissement'|'notification_client'|'followup_j3'
  payload jsonb not null default '{}',
  occurred_at timestamptz not null default now()
);
create index on activities (client_id, occurred_at desc);
```

### 4.3 Automatisations & validations

```sql
-- Journal d'exécution de chaque job — crons de l'app ET routines Claude Code
-- (les routines y insèrent leur passage via MCP Supabase ; visible dans le moniteur, §5.13)
create table automation_runs (
  id uuid primary key default gen_random_uuid(),
  job_name text not null, trigger text not null default 'cron',   -- 'cron' | 'manuel' | 'event'
  runner text not null default 'app',     -- 'app' | 'routine_claude'
  started_at timestamptz not null default now(), finished_at timestamptz,
  statut text default 'running',          -- 'running' | 'ok' | 'partial' | 'error'
  items_processed int default 0,
  tokens_in bigint default 0, tokens_out bigint default 0,  -- indicatif (couvert par le forfait Max)
  error text, log jsonb
);

-- File de validation humaine : l'IA PROPOSE, Ben DISPOSE
create table review_queue (
  id uuid primary key default gen_random_uuid(),
  kind text not null,   -- 'client_create'|'client_enrich'|'transcript_merge'|'email_link'
                        -- |'appointment_link'|'followup_j3_send'
  proposed_changes jsonb not null,        -- diff avant/après lisible, ou contenu du mail à envoyer
  source_refs jsonb,                      -- ids des mails/transcripts/RDV sources
  confidence numeric,                     -- 0..1 (score de l'IA)
  statut text default 'en_attente',       -- 'en_attente' | 'approuve' | 'rejete' | 'auto_applique'
  decided_at timestamptz,
  created_at timestamptz default now()
);

create table weekly_briefs (
  id uuid primary key default gen_random_uuid(),
  week_start date not null unique,        -- le lundi de la semaine
  content jsonb not null,                 -- par RDV : client, projet, zone, refs, extraits mails, transcript, prix
  rendered_md text,
  statut text default 'genere',           -- 'genere' | 'lu'
  generated_at timestamptz default now()
);

-- File de travail générique (découpage des gros traitements)
create table job_queue (
  id uuid primary key default gen_random_uuid(),
  kind text not null, payload jsonb not null default '{}',
  run_after timestamptz default now(), attempts int default 0, locked_at timestamptz,
  idempotency_key text unique,
  created_at timestamptz default now()
);

-- Souscriptions web push de Ben (PWA)
create table push_subscriptions (
  id uuid primary key default gen_random_uuid(),
  endpoint text unique not null, keys jsonb not null,
  user_agent text, created_at timestamptz default now()
);
```

### 4.4 Métier salon

```sql
-- Traçabilité (obligation légale : décret 2008-149 & arrêté du 6 mars 2013 —
-- traçabilité des produits de tatouage ; conservation ≥ 3 ans après usage)
create table tracabilite_sessions (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id),
  appointment_id uuid references appointments(id),
  date_seance date not null,
  praticien text not null default 'Ben',
  zone text, type_acte text not null,     -- 'tatouage' | 'piercing'
  consentement_signe boolean default false,
  photo_labels_file_id uuid references files(id),  -- photo des étiquettes de lots
  notion_page_id text unique,
  created_at timestamptz default now()
);

create table tracabilite_items (
  id uuid primary key default gen_random_uuid(),
  session_id uuid not null references tracabilite_sessions(id) on delete cascade,
  categorie text not null,                -- 'encre' | 'aiguille' | 'buse' | 'stencil' | 'autre'
  marque text, reference text,
  numero_lot text not null,
  date_peremption date,
  fournisseur text
);
-- ⚠️ Ces deux tables ne sont JAMAIS purgées, même à l'effacement RGPD d'un client
-- (pseudonymisation du client, conservation des sessions — voir §10).

create table supplier_invoices (
  id uuid primary key default gen_random_uuid(),
  fournisseur text, date_facture date,
  montant_ht numeric, montant_ttc numeric, tva numeric,
  categorie text, file_id uuid references files(id),
  statut text default 'a_extraire',       -- 'a_extraire' (work-queue routine) | 'a_valider' | 'a_payer' | 'payee'
  notes text, notion_page_id text unique,
  created_at timestamptz default now()
);

-- Consentement pré-tatouage numérique (signé depuis le portail AVANT la séance)
create table consents (
  id uuid primary key default gen_random_uuid(),
  client_id uuid not null references clients(id),
  project_id uuid references projects(id),
  appointment_id uuid references appointments(id),
  texte_version text not null,            -- version du texte de consentement (stocké dans settings)
  statut text default 'en_attente',       -- 'en_attente' | 'signe'
  signe_le timestamptz,
  signature_data jsonb,                   -- trace : nom saisi, tracé de signature, horodatage, user agent
  pdf_file_id uuid references files(id),  -- PDF généré et archivé (bucket docs)
  created_at timestamptz default now()
);
-- À la signature : tracabilite_sessions.consentement_signe passe à true pour la séance liée.

-- Suivi cicatrisation (photo demandée au client à J+15 / J+30 via le portail)
create table healing_checkins (
  id uuid primary key default gen_random_uuid(),
  project_id uuid not null references projects(id) on delete cascade,
  client_id uuid not null references clients(id),
  due_at date not null,                   -- J+15 puis J+30 après le RDV réalisé
  statut text default 'planifie',         -- 'planifie' | 'demande_envoyee' | 'photo_recue' | 'clos'
  photo_file_id uuid references files(id),
  note_client text, note_ben text,
  created_at timestamptz default now()
);
-- Créés par le cron app `cica-scheduler` ; à réception d'une photo → push à Ben (aucune analyse IA).

create table faq_items (
  id uuid primary key default gen_random_uuid(),
  categorie text not null,                -- 'tatouage' | 'piercing'
  question text not null, reponse_md text not null,
  ordre int default 0, publie boolean default true
);

-- Flashs : source de vérité pour flash.letraitnoir.fr après rapatriement (§8, étape 8)
create table flashs (
  id uuid primary key default gen_random_uuid(),
  titre text, image_file_id uuid references files(id),
  prix numeric, taille text,
  statut text default 'disponible',       -- 'disponible' | 'reserve' | 'tatoue' | 'retire'
  tags text[] default '{}',
  ar_asset_ref text,                      -- référence de l'asset AR (outil d'essayage)
  published_at timestamptz, created_at timestamptz default now()
);

-- Portfolio des réalisations
create table realisations (
  id uuid primary key default gen_random_uuid(),
  client_id uuid references clients(id) on delete set null,
  project_id uuid references projects(id) on delete set null,
  type text, style text, zone text,
  file_ids uuid[] default '{}',
  publie boolean default false, date_realisation date,
  notion_page_id text unique,
  created_at timestamptz default now()
);

-- Réglages clé/valeur (seuils IA, calendrier source, textes portail, lien avis Google, etc.)
create table settings ( key text primary key, value jsonb not null, updated_at timestamptz default now() );
```

### 4.5 Vues utiles

```sql
create view v_dashboard_today as ...   -- RDV du jour + client + projet + alertes
create view v_semaine as ...           -- RDV mardi→samedi de la semaine courante + jointures brief
```

### 4.6 Mapping Notion → Le QG (récapitulatif migration)

| Source Notion | Cible | Notes |
|---|---|---|
| Base « liste crm » (`collection://d16b8676-ed97-4e48-b947-84b9ccaa7cdf`) — propriétés | `clients` (+ `projects.zones`, `projects.cover_cicatrice`, `projects.resume`) | « Zone à tatouer », « Cover/Cicatrice », « Résumé du projet », « Rendez-vous à venir » sont des attributs de PROJET portés aujourd'hui par la fiche client → créer un projet par fiche qui en possède. Option « cuisse » du select Cover/Cicatrice = erreur de saisie à nettoyer. |
| Contenu des pages clients (« Projet Tatouage ») | `projects` + `drawings` + `files` + `projects.jalons` + `projects.devis` | Tables « Inspirations & Croquis » → `drawings` ; jalons « Suivi & prochaines étapes » → `jalons` ; images → Storage (URLs S3 Notion **expirantes** : télécharger immédiatement). |
| Relation → base « Projets » (`c3b2570e…`) | `projects` (fusion) | Base Notion simple : titre, date, fait, tâches. |
| Relation → base « Réalisations » (`1e32c757…`) | `realisations` | |
| Base « Traçabilité » (`a34a973c…`) | `tracabilite_sessions` + `tracabilite_items` | Récupérer le schéma exact au moment du script (non exploré en détail). |
| Base « Factures Fournisseurs » | `supplier_invoices` | PDFs → bucket `factures`. |
| Pages FAQ tatouage + piercing | `faq_items` | Découper en items Q/R. |
| Base « Avis Google » | **NON migrée** | Export ZIP archivé uniquement (décision Ben). |
| Flashs (autre compte Supabase) | `flashs` + bucket `flashs` | §8 étape 8 — nécessite les accès de l'autre compte. |
| Fiches de soins (PDF Google Drive) | bucket `docs` | Servies au portail en signed URL. |

---

## 5. Spécification module par module

Pour chaque module : l'app est **mobile-first ET desktop-first** (les deux comptent), états vides soignés, squelettes de chargement, aucune page à plus de 2 clics du dashboard.

### 5.1 Dashboard — « Aujourd'hui / Cette semaine » (`/`)
- **Objectif : zéro clic pour savoir quoi faire.**
- Bloc « Aujourd'hui » : RDV du jour (heure, client, type, zone), fiche dépliable inline (dernier croquis, résumé, lien fiche complète).
- Bloc « À valider » : compteur review_queue + les 3 plus récentes avec boutons Approuver/Refuser inline.
- Bloc « Alertes » : croquis à livrer avant un RDV proche, mails triés non lus, factures à payer, heartbeat Planity en erreur.
- Bloc « Brief du lundi » : lien vers le dernier brief (badge « nouveau » le lundi matin).
- Mobile : ces blocs empilés, PWA — c'est l'écran d'accueil du téléphone de Ben.

### 5.2 Clients (`/clients`, `/clients/[id]`)
- Liste : recherche instantanée (nom, prénom, email, téléphone), filtres prestation/zone, tri par dernier contact.
- Fiche client : **timeline `activities`** au centre (mails, RDV, croquis, commentaires portail, transcripts, changements de statut), infos à gauche, projets à droite.
- Gestion des liens portail : générer / copier / révoquer un token, voir dernier accès.
- Actions RGPD : export JSON du client, anonymisation (§10).
- Création manuelle rapide : modal 5 champs (nom, prénom, email, téléphone, note) — < 30 secondes.

### 5.3 Projets & croquis (`/projets`, `/projets/[id]`)
- Vue kanban par statut du pipeline (`demande → … → clos`), drag & drop.
- Fiche projet : résumé, zones, cover/cicatrice, taille, prix estimé/final, devis (lignes), jalons.
- Galerie `drawings` : upload (drag & drop + photo mobile), versions, commentaires, bascule `visible_portail` par élément.
- **Bouton « Livrer le croquis »** : publie sur le portail + déclenche la notification client automatique (checkbox « notifier le client » cochée par défaut, débrayable).

### 5.4 Agenda (`/agenda`)
- Vue semaine/mois du miroir GCal (lecture seule — la vérité reste Planity→GCal).
- Chaque RDV affiche son rapprochement : badge vert (client lié) / orange « non rapproché » → liaison en 1 clic (suggestions IA issues de review_queue, recherche manuelle sinon).
- Marquage rapide post-RDV : `realise` / `annule` / `no_show` (alimente le J+3).

### 5.5 Boîte triée (`/mails`)
- Threads groupés par triage IA : **Clients / Prospects / Fournisseurs / Planity / Plaud / Autre**. Ce n'est **pas** un client mail : lecture, triage, liaison.
- Actions : lier à un client, « créer client + projet depuis ce mail » (formulaire pré-rempli par l'extraction IA), corriger le triage (feedback stocké), **ouvrir dans Gmail** (deeplink `https://mail.google.com/mail/u/0/#inbox/<threadId>` ou mailto).
- Un clic sur l'adresse du client, partout dans l'app, ouvre Gmail.

### 5.6 Brief du lundi (`/brief`, `/brief/[week]`)
- Généré automatiquement chaque lundi 06h00 (Europe/Paris) pour la semaine mardi → samedi.
- Par RDV : bloc client (infos + lien fiche), projet (résumé, zone, taille, prix estimé), **références visuelles** (miniatures des drawings), 3 derniers échanges mail (extraits), transcript d'entretien s'il existe, champ libre « notes de préparation ».
- Historique des briefs, bouton « Régénérer », notification push à la génération.

### 5.7 Portail client (`/portail/[token]`)
- **Design galerie plein écran** (§7) — c'est la vitrine de Ben auprès de ses clients ; il doit être plus beau que la page Notion actuelle.
- Sections : accueil personnalisé (« Salut {prénom} 👋 »), résumé du projet + prochain RDV, **jalons** (projet validé / croquis livré / retouche — dérivés du statut projet), **galerie croquis** avec commentaires et bouton **« Valider ce croquis »**, documents de soins (PDF signed URL), devis.
- **Encart abonnements** : bloc promo vers abonnement.letraitnoir.fr.
- **Flashs aléatoires** : 3–4 flashs `disponible` tirés au hasard à chaque visite + lien vers flash.letraitnoir.fr.
- **Consentement pré-séance** : quand un consentement est `en_attente` pour un RDV à venir, bandeau dédié → lecture du texte + signature en ligne (nom + case cochée + tracé) → PDF généré et archivé, `tracabilite_sessions.consentement_signe` alimenté. Fini le papier.
- **Suivi cicatrisation** : à J+15 et J+30, l'espace client invite à envoyer une photo de la cicatrisation + un mot ; Ben reçoit un push à réception (aucune analyse automatique — c'est Ben qui juge).
- Sécurité : `X-Robots-Tag: noindex`, rate limiting, aucune navigation hors du token, log d'accès.

### 5.8 Traçabilité (`/tracabilite`)
- **Saisie d'une session en ≤ 60 secondes** pendant/après un RDV : pré-remplie depuis le RDV du jour (client, date, zone, type d'acte).
- Items encre/aiguilles : **autocomplete sur les derniers lots utilisés** (mémoire des saisies précédentes), photo des étiquettes au téléphone.
- Export CSV/PDF par période (contrôle DDPP). Jamais de purge.

### 5.9 Factures fournisseurs (`/factures`)
- Upload PDF (ou transfert depuis un mail fournisseur détecté) → statut `a_extraire` → **la routine Claude horaire remplit les champs** (fournisseur, date, montants) → validation par Ben dans l'app → `a_payer`/`payee`.
- Liste filtrable, totaux par mois/catégorie, export CSV comptable.

### 5.10 Flashs (`/flashs`)
- CRUD complet, statuts Disponible/Réservé/Tatoué/Retiré, tags, prix, référence asset AR.
- **Source de vérité pour flash.letraitnoir.fr** après rapatriement (P7) — la vitrine et l'outil AR liront ces tables.

### 5.11 FAQ (`/faq`)
- Éditeur markdown, deux catégories (tatouage/piercing), réordonnancement.
- Contenu réutilisable (affiché sur le portail client en accordéon, mobilisable plus tard pour des réponses assistées).

### 5.12 Centre de validations & notifications (`/validations`)
- Liste complète de la `review_queue` avec **diff visuel avant/après** pour chaque proposition, ou aperçu du mail à envoyer (J+3, notification croquis si mise en file).
- Approuver / Refuser en un tap. Historique des décisions.
- Réception push (web push PWA) à chaque nouvelle entrée + badge d'app.

### 5.13 Réglages & moniteur (`/reglages`)
- Comptes Gmail connectés (OAuth flow), calendrier GCal source, seuils de confiance IA, textes des mails transactionnels et du consentement, lien avis Google, gestion des souscriptions push.
- **Moniteur d'automatisations** : dernier run par job — crons de l'app ET routines Claude Code (`automation_runs.runner`) — vert/rouge, items traités, bouton « relancer maintenant » (crons app), logs. **Alerte si une routine Claude n'est plus passée depuis > X h** (seuil par routine dans settings).

---

## 6. Automatisations

### 6.1 Vue d'ensemble — « IA en routines Claude Max, mécanique dans l'app »

Décision structurante (validée par Ben) : **aucun appel API Claude payant**. L'app n'exécute que des tâches mécaniques et déterministes (0 token). Toute l'intelligence tourne en **routines Claude Code planifiées**, couvertes par le forfait Claude Max de Ben. Les deux mondes communiquent **exclusivement par la base de données** (tables « à traiter » dans un sens, `review_queue` + `automation_runs` dans l'autre).

```
┌────────────── APP — Vercel Cron → /api/cron/* (0 token Claude) ───────────────┐
│ gmail-sync (15 min, recopie brute) · gcal-sync (15 min, miroir + liaisons     │
│ triviales) · planity-mail-watch (15 min, parsing regex) · cica-scheduler      │
│ (quotidien) · send-outbox (envoi des mails validés) · daily-digest (07h30)    │
│ planity-heartbeat (quotidien) · backup-export (03h00)                         │
│ Événementiel : publication croquis → mail modèle · review_queue → push        │
└──────────────────────────┬───────────────────▲────────────────────────────────┘
        tables « à traiter » (statuts)   review_queue + automation_runs
┌──────────────────────────▼───────────────────┴────────────────────────────────┐
│ ROUTINES CLAUDE CODE — forfait Max, déclencheurs cron (minimum 1 h)           │
│ mail-triage (horaire) · transcript-ingest (horaire) · facture-extraction      │
│ (horaire) · followup-j3 (quotidien 08h00) · brief-hebdo (lundi 06h00)         │
│ tattooversaire (lundi 07h00) · pont Planity→GCal (existant) · ad hoc          │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Principes transverses (NON NÉGOCIABLES)

1. **Idempotence** : clés naturelles uniques partout (`gmail_message_id`, `gcal_event_id`, `idempotency_key`) ; tout en upsert ; chaque routine filtre par statut « à traiter » et met à jour le statut en fin de traitement — un passage peut être rejoué sans dommage. Test d'acceptation : 3 rejeux forcés → zéro doublon.
2. **Découpage** : chaque tick/passage borne son travail (ex. 25 threads max) ; le reste attend le passage suivant. Jamais de timeout.
3. **L'IA propose, Ben dispose** : auto-application UNIQUEMENT pour les liaisons triviales (adresse email exactement égale à celle d'un client existant → lier le thread — fait par l'APP, de façon déterministe). Tout le reste (création client, enrichissement de champs, fusion transcript, envoi J+3, tattooversaire) passe en `review_queue` + push.
4. **Traçage** : chaque passage loggé dans `automation_runs` (runner 'app' ou 'routine_claude', statut, items). Crons app : retry exponentiel ×3 via `job_queue.attempts`. Routines : le moniteur alerte si aucun run récent (seuil par routine).
5. **Latences assumées** : ≤ 15 min pour la mécanique (crons app), ≤ 1 h pour l'intelligence (intervalle minimum des routines). Aucun traitement urgent ne dépend d'une routine.
6. **Frais** : zéro coût API variable. Si une routine devient trop gourmande pour les limites du forfait Max, la réduire en fréquence ou en périmètre (jamais basculer silencieusement sur l'API payante).

### 6.3 Crons de l'app (mécaniques, 0 token Claude)

| Job | Cron (Europe/Paris) | Logique |
|---|---|---|
| `gmail-sync` | */15 min | Par compte Gmail actif : `history.list` depuis `last_history_id` → upsert `email_threads`/`email_messages`. **Recopie brute, aucune analyse.** Règles déterministes au passage : expéditeur = client connu → liaison auto + `triage='client_connu'` ; expéditeur Planity → `triage='planity'` ; expéditeur Plaud → `triage='plaud'` + création `transcripts` (statut `a_valider`). Le reste reste `nouveau` (work-queue de la routine mail-triage). Premier run : backfill borné (90 jours). |
| `gcal-sync` | */15 min | `events.list` incrémental (`syncToken`) → upsert `appointments`. Liaison déterministe : nom du titre exactement égal à un client unique → lien auto ; sinon laissé « non rapproché » (suggestions par la routine mail-triage ou liaison manuelle). RDV passés non marqués → `realise` par défaut à J+1 (corrigeable). |
| `planity-mail-watch` | */15 min | Threads `triage='planity'` : parsing **regex** des mails de confirmation (format templaté par Planity) → si aucun `appointment` correspondant à ±15 min n'existe 24 h après → alerte « RDV Planity absent du calendrier » (2e ceinture du pont fragile). |
| `planity-heartbeat` | quotidien 20h00 | Si aucun événement GCal créé/modifié depuis N jours ouvrés (settings, défaut 3) → push « Le pont Planity semble muet ». |
| `cica-scheduler` | quotidien 09h00 | Pour chaque RDV `realise` (tatouage) : crée les `healing_checkins` J+15 et J+30 s'ils n'existent pas ; à `due_at`, envoie le mail modèle « comment va la cicatrisation ? » avec le lien portail (`demande_envoyee`). À réception d'une photo → push à Ben. |
| `send-outbox` | */15 min | Envoie via Gmail API les mails **validés** en attente : J+3 approuvés, tattooversaires approuvés. Templates versionnés, ton de Ben. Log `activities` + statuts. |
| `daily-digest` | quotidien 07h30 | Mail à Ben : RDV du jour, validations en attente, alertes, état des jobs ET des routines. (Fallback des push.) |
| `backup-export` | quotidien 03h00 | Export logique des tables métier (CSV/JSON) → bucket `backups`, rotation 30 jours. En plus des backups Supabase Pro. |

**Événementiel (app, hors cron)** — notification croquis : quand Ben publie un drawing (`visible_portail=true`) avec la case « notifier le client » cochée → vérifier/créer un token portail actif → envoyer le mail **modèle** (Gmail API, ton de Ben) : « Ça y est, Ben a ajouté un dessin pour ton projet. Tu peux le découvrir ici → [lien privé] » → logger `activities` + `drawings.notification_envoyee_at`. Anti-spam : max 1 notification par projet par heure (regroupement). Zéro IA : c'est un template.

### 6.4 Routines Claude Code (forfait Max — l'intelligence)

Chaque routine est créée avec un déclencheur cron (minimum 1 h), lit ses instructions versionnées dans `routines/<nom>.md` du repo, accède à la base via le **MCP Supabase**, et suit le contrat §6.5.

| Routine | Déclencheur | Lit (work-queue) | Écrit | Rôle |
|---|---|---|---|---|
| `mail-triage` | horaire | `email_threads` `triage='nouveau'` (+ messages) | `email_threads.triage`, `review_queue` (`client_create`/`client_enrich`/`email_link`), `automation_runs` | Classifie (prospect/fournisseur/perso/spam), extrait {nom, prénom, projet, zone, taille, budget, dispo} des demandes de tatouage, propose créations/enrichissements avec confidence. Suggère aussi les rapprochements de RDV « non rapprochés ». |
| `transcript-ingest` | horaire | `transcripts` `statut='a_valider'` | `transcripts.resume_ia/extracted`, `review_queue` (`transcript_merge`), `automation_runs` | Résume l'entretien Plaud, extrait les infos structurées, propose la fusion dans la fiche client/projet (diff avant/après). |
| `facture-extraction` | horaire | `supplier_invoices` `statut='a_extraire'` | champs de la facture, `statut='a_valider'`, `automation_runs` | Lit le PDF, remplit fournisseur/date/montants/TVA/catégorie. Ben valide dans l'app. |
| `followup-j3` | quotidien 08h00 | `appointments` `realise` à J-3 avec `followup_j3_statut='na'` | `review_queue` (`followup_j3_send` avec le mail rédigé), `followup_j3_statut='propose'`, `automation_runs` | Rédige le mail personnalisé « tout va bien après ton passage ? » (ton de Ben, lien avis Google). L'envoi est fait par l'app (`send-outbox`) APRÈS validation. |
| `brief-hebdo` | lundi 06h00 | `v_semaine` + jointures (projets, drawings, 3 derniers mails, transcripts) | `weekly_briefs`, `automation_runs` | Compile et rédige le brief du lundi par RDV. L'app détecte le nouveau brief → push « Ton brief du lundi est prêt ». |
| `tattooversaire` | lundi 07h00 | projets `realise` il y a ~1 an (fenêtre de la semaine) | `review_queue` (brouillon de mail anniversaire), `automation_runs` | Propose le mail « tattooversaire » personnalisé. Envoi via `send-outbox` après validation. |
| `pont Planity→GCal` | existant (inchangé) | Planity via Chrome | Google Calendar | Recopie l'agenda Planity dans GCal. Surveillé par `planity-heartbeat` + `planity-mail-watch`. |

### 6.5 Contrat d'interface app ↔ routines (`routines/CONTRAT.md`)

1. Les routines n'écrivent **JAMAIS** directement dans `clients`/`projects` : toute modification de fiche passe par `review_queue.proposed_changes` au format `{ table, id?, before, after, justification, confidence }`. Exceptions listées : `email_threads.triage`, `transcripts.resume_ia/extracted`, `supplier_invoices` (champs extraits + statut `a_valider`), `weekly_briefs`.
2. Chaque routine **ouvre** un `automation_runs` (runner `routine_claude`) en début de passage et le **clôt** (statut, items) en fin — c'est ce qui alimente le moniteur et les alertes de silence.
3. Idempotence par statut : ne traiter que les lignes « à traiter », les faire changer de statut, ne jamais retraiter une ligne déjà passée.
4. Passage borné (25 items max) ; le reste attend l'heure suivante.
5. Jamais de contenu envoyé à un client par une routine : la rédaction oui, l'envoi non — l'envoi est toujours fait par l'app après validation de Ben (sauf notification croquis, qui est un template app déclenché par Ben lui-même).
6. Interdiction d'extraire des données médicales/santé (voir §10) — si un mail ou transcript en contient, le signaler à Ben sans les structurer.

### 6.6 Le pont Planity (rappel du contrat)

Planity n'a pas d'API. La routine Claude Code existante (navigation Chrome) recopie l'agenda Planity dans Google Calendar. **Le QG ne parle jamais à Planity directement** ; il lit Google Calendar et surveille le pont via `planity-heartbeat` + `planity-mail-watch`. Si le pont casse, Ben est alerté mais l'app continue de fonctionner.

---

## 7. Design system « Encre & papier »

### 7.1 Direction artistique

L'app doit ressembler à ce que Ben fait : **du trait noir sur une surface claire, et du blanc qui claque sur du noir profond**. Référence mentale : une galerie d'art contemporaine croisée avec un carnet de croquis. Niveau d'exigence : **AWWWards Site of the Day**.

- **Palette** : noir encre `#0A0A0A`, blanc papier `#F7F5F2`, gamme de gris chauds intermédiaires. **Un seul accent** utilisé avec parcimonie (rouge sang très désaturé `#8C2B2B` ou laisser en N&B strict — à trancher en Phase 0 avec 2 propositions). App interne majoritairement sombre ; **portail client majoritairement papier clair** (l'encre des croquis ressort sur fond clair).
- **Typographie** : une **serif display expressive** pour les titres (ex. « Editorial New »-like en fonte libre : *Instrument Serif*, *Fraunces*…) + une **grotesque sobre** pour l'UI (*Inter*, *General Sans*…). Corps généreux, hiérarchie marquée, chiffres tabulaires pour les données.
- **Textures & détails** : grain papier subtil (SVG noise), traits d'encre comme séparateurs (irréguliers, dessinés à la main, en SVG), ombres douces, coins à peine arrondis.
- **Motion** (Framer Motion) : transitions de page fluides (fade + léger déplacement), micro-interactions sur hover/tap, apparition des listes en cascade discrète, `prefers-reduced-motion` respecté. Le motion souligne, il ne distrait jamais.
- **Iconographie** : un seul set cohérent, trait fin (Lucide personnalisé ou set custom), jamais d'emoji dans l'UI de production.

### 7.2 Exigences techniques design

- **Responsive irréprochable** : breakpoints mobile/tablette/desktop ; l'app interne est utilisée au salon sur téléphone (traçabilité, validations) et le lundi sur desktop (brief, dessins).
- **PWA** : manifest, icônes, splash, service worker (cache app shell + web push). Installable iOS/Android.
- **Performance** : Lighthouse ≥ 90 sur les 4 axes, portail comme app. Images optimisées (`next/image`), fontes self-hosted avec `font-display: swap`.
- **Accessibilité** : contrastes AA minimum (facile en N&B), focus visibles, navigation clavier.
- **Composants du design system** (Phase 0) : boutons, inputs, cards, table/liste, kanban card, timeline item, modal, toast, badge de statut (un style par statut du pipeline), empty states illustrés (petits dessins au trait), skeletons.

---

## 8. Plan de migration Notion

Scripts **one-shot idempotents** dans `scripts/migration-notion/` (exécutés en local avec l'API Notion + service role Supabase). Chaque script est rejouable : clé `notion_page_id`, upserts, dédoublonnage fichiers par `sha256`. Ben a validé une approche **étape par étape**.

**Ordre d'exécution :**
1. **Référentiels** : `body_zones` (les ~24 zones du multi-select), FAQ → `faq_items`, fiches de soins PDF (Google Drive) → bucket `docs`.
2. **Clients** : base « liste crm » → `clients` (mapping §4.6). Nettoyages : option « cuisse » du select Cover/Cicatrice (erreur de saisie), doublons d'email, normalisation téléphones.
3. **Projets** : propriétés projet portées par la fiche client + base « Projets » liée → `projects` (un projet par fiche qui a une zone/un résumé/un RDV). Parsing du contenu des pages : résumé, jalons, devis.
4. **Fichiers** : toutes les images/croquis des pages et tables Notion. ⚠️ **Les URLs de fichiers Notion (S3) sont expirantes** : télécharger immédiatement à la lecture de chaque page, uploader vers Storage, créer `files` + `drawings`. Dédoublonnage `sha256`.
5. **Traçabilité** : base Notion → `tracabilite_sessions`/`tracabilite_items` (relever le schéma exact de la base au moment du script).
6. **Factures fournisseurs** : lignes + PDFs → `supplier_invoices` + bucket `factures`.
7. **Réalisations** : base Notion → `realisations` + fichiers.
8. **Flashs** : depuis **l'autre compte Supabase** (celui de flash.letraitnoir.fr) → `flashs` + bucket `flashs`. Nécessite les credentials de l'autre compte (à fournir par Ben). Rebranchement de la vitrine et de l'outil AR en P7.
9. **Backfill mails 12 mois** (optionnel mais fortement recommandé) : recherche Gmail par adresse de chaque client migré → `email_threads`/`email_messages` liés → la timeline est riche dès le premier jour.

**Validation :**
- Rapport de migration automatique : comptages Notion vs Supabase par table, liste des pages non parsées, fichiers en échec.
- Revue manuelle par Ben : 10 fiches clients représentatives + 3 pages portail comparées côte à côte avec la page Notion partagée équivalente.

**Parallel run — 4 semaines :**
- Dès la bascule : Notion passe en **lecture seule** (plus aucune saisie), Le QG est la source de vérité.
- J+7 : re-run des scripts en mode diff pour rattraper les oublis.
- Résiliation Notion seulement après : (a) 4 semaines sans manque constaté, (b) **export ZIP officiel Notion archivé** dans le bucket `backups`.

---

## 9. Plan de livraison par phases

Chaque phase est un incrément **déployable et démontrable à Ben**. Ne pas commencer une phase avant que les critères de la précédente soient verts.

| Phase | Contenu | Critères d'acceptation |
|---|---|---|
| **P0 — Socle** | Repo `le-qg`, Next.js + Tailwind + design system « Encre & papier » (tokens, typo, composants de base), migrations SQL complètes (§4), auth Ben + TOTP, PWA (manifest + SW + push souscription), déploiement `qg.letraitnoir.fr` | Login Ben (avec MFA) fonctionne ; dashboard vide stylé déployé en prod ; PWA installable sur le téléphone de Ben avec une push de test reçue ; RLS vérifiée (anon = zéro accès sur toutes les tables) |
| **P1 — CRM & projets (manuel)** | Modules Clients, Projets/kanban, drawings upload, timeline, recherche, Traçabilité (saisie manuelle) | Créer client + projet + croquis en < 1 min ; timeline agrège les événements ; saisie traçabilité ≤ 60 s ; tout utilisable au téléphone |
| **P2 — Migration Notion** | Scripts étapes 1→7 (§8), rapport de validation | Comptages égaux ; 10 fiches validées par Ben ; toutes les images visibles ; zéro doublon après rejeu des scripts |
| **P3 — Portail client** | Pages tokenisées, galerie, commentaires + validation croquis, docs de soins, encart abonnements, flashs aléatoires, notification croquis (event), **consentement numérique signé**, **upload photo cicatrisation** | 3 pages portail jugées « mieux que Notion » par Ben ; token révocable ; notification croquis reçue en test réel ; un consentement signé génère son PDF archivé ; Lighthouse mobile ≥ 90 ; noindex vérifié |
| **P4 — Syncs mécaniques** | gcal-sync, gmail-sync (recopie brute + règles déterministes), boîte triée, rapprochements manuels, review_queue + push, moniteur, planity-heartbeat + planity-mail-watch, cica-scheduler, send-outbox | Un RDV créé dans Planity apparaît < 15 min après le passage du pont ; mail d'un client connu lié automatiquement (règle déterministe) ; 3 rejeux forcés sans doublon ; push de validation reçue et actionnable depuis le téléphone |
| **P5 — Routines IA (Claude Max)** | Création des routines `mail-triage`, `transcript-ingest`, `followup-j3`, `facture-extraction` + fichiers `routines/*.md` + contrat d'interface + alertes de silence au moniteur | Mail inconnu → proposition de création en ≤ 1 h, validable en 1 tap ; mail Plaud → transcript → proposition de fusion ; J+3 rédigé le bon jour et JAMAIS envoyé sans validation ; chaque routine visible au moniteur avec son dernier passage ; zéro appel API payant vérifié |
| **P6 — Brief du lundi** | Routine `brief-hebdo` (lundi 06h00) + page brief + push à la détection | Lundi 07h00 : un brief complet est visible dans l'app pour chaque RDV de la semaine (client, projet, zone, refs, extraits mails, transcript, prix) ; Ben le juge utilisable tel quel deux lundis de suite |
| **P7 — Modules métier restants** | Factures (work-queue + routine), FAQ, routine `tattooversaire`, Flashs : migration depuis l'autre compte Supabase + rebranchement de flash.letraitnoir.fr et de l'outil AR, re-pointage du skill `ajout-flashs` | Facture PDF uploadée → champs remplis par la routine en ≤ 1 h → validée en < 30 s ; flash publié dans Le QG visible sur la vitrine ; essayage AR fonctionne ; export traçabilité CSV conforme |
| **P8 — Parallel run & extinction** | 4 semaines de double-run (§8), backfill mails, corrections, export ZIP Notion archivé, démantèlement/re-pointage des routines et skills existants (brief-semaine, triage matinal), résiliation Notion & Spark | 4 semaines sans retour à Notion ; checklist RGPD/backups verte ; aucune double-écriture résiduelle |

---

## 10. Sécurité & RGPD

- **Données sensibles** : le contexte tatouage (cicatrices, covers, zones corporelles, contre-indications éventuelles dans les transcripts) peut relever des **données de santé**. Minimisation : ne JAMAIS demander à l'IA (routines Claude comprises — c'est dans le contrat §6.5) d'extraire des données médicales ; tout champ santé est saisi manuellement par Ben. Bases légales : exécution du contrat + obligation légale (traçabilité).
- **Mineurs** : si `date_naissance` → âge < 18 ans, bandeau d'alerte visuel sur la fiche + champ « autorisation parentale » requis avant statut `rdv_planifie`.
- **Portail** : tokens 256 bits générés par CSPRNG, stockés **hashés avec pepper**, rate limiting sur `/portail/*`, expiration optionnelle, journal d'accès, `X-Robots-Tag: noindex`, signed URLs Storage ≤ 1 h, aucune donnée d'un autre client accessible.
- **Secrets** : refresh tokens Google dans Supabase Vault / env Vercel. Service role key **jamais** exposée côté client. RLS deny-by-default sur toutes les tables et sur Storage.
- **Rétention** : traçabilité ≥ 3 ans, jamais purgée. Droit à l'effacement : anonymisation du client (nom → « Client supprimé », PII effacées, mails détachés) en **préservant** les sessions de traçabilité pseudonymisées. Droit à la portabilité : export JSON par client (bouton sur la fiche).
- **Backups** : double ceinture — Supabase Pro (daily + PITR) **et** export logique quotidien vers le bucket `backups` (rotation 30 j). Test de restauration à faire une fois en P8.
- **Registre des traitements** de Ben à mettre à jour ; mentions d'information sur le portail (responsable de traitement, finalités, durées, droits).

---

## 11. Risques & questions ouvertes

| # | Sujet | État / action |
|---|---|---|
| 1 | **Accès à l'autre compte Supabase (flashs)** | Confirmé par Ben que les flashs y vivent. Obtenir les credentials avant P7 ; définir la fenêtre de bascule de flash.letraitnoir.fr et de l'outil AR. |
| 2 | **Pont Planity fragile** (navigation Chrome) | Mitigé par la double ceinture (`planity-heartbeat` + `planity-mail-watch`). Accepté : si le pont casse, alerte, pas de perte de données. |
| 3 | **OAuth Google** : une app GCP en mode « testing » a des refresh tokens qui expirent à 7 jours | À traiter en P4 : publier l'app OAuth (vérification Google si scopes sensibles Gmail). Ben : « à voir » — prévoir ce point comme blocant potentiel de P4. |
| 4 | **Budget récurrent** : Vercel Pro (~20 $/mois) + Supabase Pro (~25 $/mois). **Zéro frais API variable** : l'IA tourne en routines sur le forfait Claude Max de Ben | Compensé par la résiliation de Notion + Spark. |
| 5 | **Parsing des pages Notion hétérogènes** | Approche étape par étape validée ; le rapport de migration liste les cas à traiter à la main. |
| 6 | **Canal des mails transactionnels** : Gmail API send (recommandé — part de la vraie adresse de Ben, ton personnel) vs Brevo (délivrabilité trackée) | Trancher en P3 au premier envoi réel. |
| 7 | **Push iOS** : nécessite l'installation de la PWA sur l'écran d'accueil (iOS ≥ 16.4) | Documenter dans l'onboarding ; fallback daily-digest 07h30. |
| 8 | **Dépendance aux routines Claude Code** : disponibilité du service, limites d'usage du forfait Max si les routines deviennent gourmandes, latence minimum 1 h | Assumé (décision Ben). Mitigation : moniteur avec alerte de silence par routine, passages bornés (25 items), daily-digest en fallback. Jamais de bascule silencieuse vers l'API payante. |
| 9 | **Démantèlement de l'existant** : routines/skills actuels (tri matinal, brief-semaine, ajout-flashs) | À re-pointer vers Le QG ou éteindre en fin de P8 pour éviter les doubles-écritures. Le pont Planity→GCal, lui, RESTE. Les nouvelles routines §6.4 REMPLACENT les anciennes. |

---

## 12. Backlog

**Backlog vidé — tout a été arbitré avec Ben le 2026-07-20 :**
- **Intégrés au périmètre** (voir §2.11, §4.4, §5.7, §6.4, §9) : consentement pré-tatouage numérique (P3), suivi cicatrisation J+15/J+30 (P3 + P4), tattooversaire (P7).
- **Écartés définitivement** : mini-stats (Planity les fournit), relance des projets sans réponse, stock consommables (Planity le couvre). Ne pas les proposer à nouveau sans demande de Ben.

---

*Fin de la fiche projet. Pour toute ambiguïté pendant le build : demander à Ben plutôt que supposer — en particulier sur le périmètre portail, les envois de mails aux clients et tout ce qui touche aux données de santé.*
