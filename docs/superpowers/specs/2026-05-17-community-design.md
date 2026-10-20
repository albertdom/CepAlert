# CepAlert — Comunitat i dades pròpies (Feature D)

**Data:** 2026-05-17
**Estat:** Aprovat
**Sessió:** continuació de brainstorming post-MVP

## Objectiu

Afegir una capa de confirmacions reals d'usuaris al mapa existent. Els usuaris poden marcar zones on han trobat ceps, i la comunitat pot corroborar o desmentir cada confirmació, generant un trust ratio per usuari. El scoring offline no canvia — la comunitat és una capa visual a sobre.

Hipòtesi a validar: ¿els usuaris confirmen trobades, i les corroboracions redueixen el soroll?

---

## Decisions de disseny

| Decisió | Elecció | Motiu |
|---|---|---|
| Backend | Supabase (hosted) | Zero DevOps, PostGIS natiu, free tier aguanta milers d'usuaris, Auth + Storage inclosos |
| Autenticació | Google Sign-In via Supabase Auth | Un tap, identitat real per trust ratio meaningful, sense gestió de contrasenyes |
| Scoring offline | No canvia | La comunitat és opcional i addictiva; si Supabase cau, l'app segueix funcionant |
| Seeding inicial | iNaturalist API (research-grade) | GPS de telèfon (3-15m), verificació independent × 2, llicència CC, API gratuïta |
| Fotos | Supabase Storage, opcional | Augmenta credibilitat però no bloqueja la confirmació |

---

## Arquitectura

```
Android app
  ├── Supabase Auth   →  Google Sign-In
  ├── Supabase DB     →  PostgreSQL + PostGIS
  │     ├── users
  │     ├── confirmations
  │     └── corroborations
  └── Supabase Storage  →  fotos (bucket privat, accés per URL signada)

Seeding (script Python one-time)
  └── iNaturalist API  →  taxon_id=54791, quality_grade=research, place=Catalunya
                       →  INSERT confirmations (source='inat', trust_ratio=1.0)
```

L'app parla directament amb Supabase via SDK Android — no cal API pròpia.

---

## Model de dades

### `users`
| Camp | Tipus | Notes |
|---|---|---|
| `id` | uuid PK | = Supabase Auth UID |
| `display_name` | text | del compte Google |
| `trust_ratio` | float | 0.0–1.0, comença a 0.5 |
| `confirmation_count` | int | total de confirmacions fetes |
| `accepted_count` | int | confirmacions corroborades positivament |

### `confirmations`
| Camp | Tipus | Notes |
|---|---|---|
| `id` | uuid PK | |
| `user_id` | uuid FK | NULL per a seeds iNaturalist |
| `lat` | float | coordenada GPS |
| `lon` | float | coordenada GPS |
| `species` | text | 'boletus_edulis' per defecte |
| `photo_url` | text? | URL signada Supabase Storage |
| `quantity` | text? | 'few' \| 'many' |
| `source` | text | 'user' \| 'inat' |
| `created_at` | timestamptz | |

### `corroborations`
| Camp | Tipus | Notes |
|---|---|---|
| `confirmation_id` | uuid FK | |
| `user_id` | uuid FK | no pot ser igual a confirmation.user_id |
| `vote` | text | 'up' \| 'down' |
| `created_at` | timestamptz | |

Constraint: `UNIQUE(confirmation_id, user_id)` — un usuari, un vot per confirmació.

---

## Trust ratio

- Nou usuari: **0.5**
- Corroboració positiva rebuda: `+0.05` (màxim 1.0)
- Corroboració negativa rebuda: `−0.08` (mínim 0.0) — penalitza més per disuadir spam
- Seeds iNaturalist: `trust_ratio = 1.0` (research-grade)
- Un usuari no pot corroborar les seves pròpies confirmacions (enforced DB + client)
- El trust ratio pondera la visibilitat de les confirmacions al mapa

---

## Flux d'usuari

### 1. Fer una confirmació
1. Tap llarg sobre qualsevol punt del mapa → bottom sheet
2. Botó principal: **"He trobat ceps aquí"**
3. Opcionals: foto (càmera/galeria), quantitat (pocs/molts)
4. Si no autenticat → Google Sign-In primer (un tap), torna al flow
5. Es guarda localment si no hi ha xarxa → WorkManager reintenta quan hi ha connexió

### 2. Veure confirmacions d'una zona
El detail sheet existent afegeix:
- Badge **"N confirmacions"** amb data de la més recent
- Llista: avatar + data + trust ratio visual + foto si n'hi ha
- Botó **"Jo també n'he trobat"** / **"No n'he vist"** per corroborar

### 3. Capa de comunitat al mapa
Toggle nou (com el de vegetació ICGC) mostra punts de calor on hi ha confirmacions de les **últimes 2 setmanes**. Color = densitat de confirmacions.

---

## Regles de negoci

- No es pot confirmar la mateixa zona dues vegades en **7 dies**
- Confirmacions de fa **+6 mesos** es marquen com "Observació histórica"
- Seeds iNaturalist mostren: *"Observació histórica · Font: iNaturalist"*
- La capa de comunitat és opcional: si Supabase no és accessible, el badge desapareix i l'app funciona normalment (offline scoring intacte)

---

## Seeding iNaturalist

Script Python one-time (`tools/seed_community/seed_inat.py`):
1. Crida iNaturalist API: `taxon_id=54791`, `quality_grade=research`, `place_id=` (Catalunya)
2. Filtra per precisió GPS `positional_accuracy < 50` metres
3. INSERT a `confirmations` amb `source='inat'`, `user_id=NULL`, coordenades reals
4. Atribució: camp `source_label = 'iNaturalist (CC BY-NC)'`

---

## Gestió d'errors

| Escenari | Comportament |
|---|---|
| Supabase offline | App funciona normalment; capa comunitat oculta amb missatge discret |
| Confirmació sense xarxa | Guardada localment, WorkManager reintenta |
| Foto fallida | Confirmació es crea sense foto; es pot afegir després |
| Auth revocada | Redirigeix a Sign-In en el pròxim intent d'acció |

---

## Testing

- **Unit:** càlcul de trust ratio (pujades, baixades, límits 0/1)
- **Integration:** Supabase local emulator per Auth + DB (PostgREST)
- **Manual:** flux complet confirmació → corroboració → canvi de trust ratio

---

## Pendents i decisions futures

- **Notificacions push** (quan una zona propera supera un llindar): requereix FCM + Play Store publicat
- **Historial personal**: llista de les meves confirmacions — UX B
- **Mapa de calor agregat** (totes les confirmacions, no solo recents): requereix més dades
- **Nivells de reputació** (novell → expert → verificat): descartada ara per sobre-enginyeria; reconsiderar quan hi hagi dades reals
- **Moderació**: no cal ara; trust ratio baix és suficient filtre inicial
