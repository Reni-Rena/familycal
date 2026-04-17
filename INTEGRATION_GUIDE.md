# Guide d'intégration — FamilyCal
## Connecter Google Calendar, Outlook et les calendriers scolaires

---

## 1. Google Calendar

### Méthode simple : lien .ics (lecture seule)

C'est la méthode la plus rapide, aucune clé API nécessaire.

1. Ouvrir **calendar.google.com**
2. À gauche, cliquer sur ⋮ à côté du calendrier voulu → **Paramètres et partage**
3. Descendre jusqu'à **Intégrer le calendrier**
4. Copier **l'adresse secrète au format iCal** (lien .ics)
5. Dans `config.js` (voir ci-dessous), ajouter ce lien

### Méthode avancée : API Google Calendar (lecture + écriture)

#### Étapes de configuration :

1. Aller sur https://console.cloud.google.com
2. Créer un projet → Activer l'**API Google Calendar**
3. Créer des identifiants OAuth 2.0 (type : Application Web)
4. Ajouter `http://localhost:8080` et votre domaine dans les origines autorisées
5. Copier le **Client ID**

#### Code d'intégration (à ajouter dans index.html) :

```html
<!-- Dans <head> -->
<script src="https://apis.google.com/js/api.js"></script>
<script src="https://accounts.google.com/gsi/client"></script>

<script>
const GOOGLE_CLIENT_ID = 'VOTRE_CLIENT_ID.apps.googleusercontent.com';
const CALENDAR_SCOPE = 'https://www.googleapis.com/auth/calendar.readonly';

let tokenClient;

function initGoogleAuth() {
  tokenClient = google.accounts.oauth2.initTokenClient({
    client_id: GOOGLE_CLIENT_ID,
    scope: CALENDAR_SCOPE,
    callback: (response) => {
      if (response.error) return;
      fetchGoogleEvents(response.access_token);
    },
  });
}

async function fetchGoogleEvents(token) {
  const now = new Date().toISOString();
  const future = new Date(Date.now() + 60*24*60*60*1000).toISOString(); // +60 jours

  const res = await fetch(
    `https://www.googleapis.com/calendar/v3/calendars/primary/events?timeMin=${now}&timeMax=${future}&singleEvents=true&orderBy=startTime`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  const data = await res.json();

  // Convertir au format FamilyCal
  const events = data.items.map(item => ({
    id: item.id,
    date: (item.start.date || item.start.dateTime).substring(0, 10),
    title: item.summary,
    type: 'perso',  // ou détecter par calendrier
    time: item.start.dateTime
      ? new Date(item.start.dateTime).toLocaleTimeString('fr-FR', {hour:'2-digit', minute:'2-digit'})
      : '',
    source: 'Google Calendar',
    note: item.description || '',
  }));

  // Ajouter à EVENTS global
  EVENTS = [...EVENTS.filter(e => e.source !== 'Google Calendar'), ...events];
  render();
}

// Bouton de connexion
document.getElementById('btnGoogleLogin').addEventListener('click', () => {
  tokenClient.requestAccessToken();
});

window.onload = () => {
  gapi.load('client', initGoogleAuth);
};
</script>
```

---

## 2. Microsoft Outlook / Microsoft 365

### Méthode simple : lien .ics

1. Ouvrir **outlook.live.com** ou Outlook 365
2. Paramètres (engrenage) → **Afficher tous les paramètres Outlook**
3. Calendrier → **Calendriers partagés**
4. Publier un calendrier → Copier le lien **ICS**

### Méthode avancée : Microsoft Graph API

1. Aller sur https://portal.azure.com → **Azure Active Directory**
2. Enregistrer une application → type : SPA
3. Permissions déléguées : `Calendars.Read`
4. Copier le **Client ID** et **Tenant ID**

#### Code d'intégration :

```html
<script src="https://alcdn.msauth.net/browser/2.30.0/js/msal-browser.min.js"></script>

<script>
const msalConfig = {
  auth: {
    clientId: 'VOTRE_CLIENT_ID',
    authority: 'https://login.microsoftonline.com/common',
    redirectUri: window.location.origin,
  }
};

const msalInstance = new msal.PublicClientApplication(msalConfig);
const GRAPH_SCOPE = ['Calendars.Read'];

async function loginOutlook() {
  try {
    const loginResponse = await msalInstance.loginPopup({ scopes: GRAPH_SCOPE });
    const tokenResponse = await msalInstance.acquireTokenSilent({
      scopes: GRAPH_SCOPE,
      account: loginResponse.account,
    });
    fetchOutlookEvents(tokenResponse.accessToken);
  } catch (e) {
    console.error('Erreur login Outlook', e);
  }
}

async function fetchOutlookEvents(token) {
  const now = new Date().toISOString();
  const future = new Date(Date.now() + 60*24*60*60*1000).toISOString();

  const res = await fetch(
    `https://graph.microsoft.com/v1.0/me/calendarView?startDateTime=${now}&endDateTime=${future}&$select=subject,start,end,bodyPreview&$orderby=start/dateTime`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  const data = await res.json();

  const events = data.value.map(item => ({
    id: item.id,
    date: item.start.dateTime.substring(0, 10),
    title: item.subject,
    type: 'perso',
    time: new Date(item.start.dateTime).toLocaleTimeString('fr-FR', {hour:'2-digit', minute:'2-digit'}),
    source: 'Outlook',
    note: item.bodyPreview || '',
  }));

  EVENTS = [...EVENTS.filter(e => e.source !== 'Outlook'), ...events];
  render();
}
</script>
```

---

## 3. Calendriers scolaires (.ics — Pronote, EcoleDirecte, ENT)

### Comment récupérer le lien .ics

#### Pronote
1. Connexion sur votre ENT → Pronote
2. Menu **Emploi du temps** → Exporter
3. Copier le **lien iCalendar** (format .ics)

#### EcoleDirecte
1. Connexion → **Agenda** → Icône calendrier
2. **Exporter** → Copier le lien .ics

### Lecture d'un fichier .ics en JavaScript

```javascript
// Coller dans config.js
const ICS_SOURCES = [
  {
    url: 'https://www.VOTREECOLE.fr/calendrier.ics',
    type: 'enfant',
    label: 'Emploi du temps Emma',
  },
  {
    url: 'https://pronote.VOTREECOLE.fr/ics/eleve.ics?token=XXXX',
    type: 'reunion',
    label: 'Réunions parents',
  },
];

// Fonction de chargement
async function loadIcsSources() {
  for (const src of ICS_SOURCES) {
    try {
      // ⚠️ Nécessite un proxy CORS ou que le serveur autorise les requêtes cross-origin
      // Option 1 : proxy local (Node.js) — voir proxy.js
      // Option 2 : corsproxy.io pour les tests : https://corsproxy.io/?URL
      const res = await fetch(`https://corsproxy.io/?${encodeURIComponent(src.url)}`);
      const text = await res.text();
      const parsed = parseIcs(text, src.type, src.label);
      EVENTS = [...EVENTS.filter(e => e.source !== src.label), ...parsed];
      render();
    } catch(e) {
      console.warn(`Impossible de charger ${src.label}:`, e);
    }
  }
}

// Parser .ics minimaliste
function parseIcs(icsText, type, source) {
  const events = [];
  const blocks = icsText.split('BEGIN:VEVENT');
  for (let i = 1; i < blocks.length; i++) {
    const block = blocks[i];
    const get = (key) => {
      const m = block.match(new RegExp(key + '[^:]*:([^\\r\\n]+)'));
      return m ? m[1].trim() : '';
    };
    const dtstart = get('DTSTART');
    if (!dtstart) continue;
    const date = dtstart.length >= 8
      ? `${dtstart.substring(0,4)}-${dtstart.substring(4,6)}-${dtstart.substring(6,8)}`
      : '';
    const time = dtstart.length > 8
      ? `${dtstart.substring(9,11)}h${dtstart.substring(11,13)}`
      : '';
    events.push({
      id: 'ics_' + Math.random().toString(36).slice(2),
      date,
      title: get('SUMMARY'),
      type,
      time,
      source,
      note: get('DESCRIPTION'),
    });
  }
  return events;
}
```

---

## 4. Configuration centralisée (config.js)

Créer un fichier `config.js` à côté de `index.html` :

```javascript
// config.js — Toutes vos sources en un seul endroit

window.APP_CONFIG = {

  // Google Calendar
  google: {
    enabled: true,
    clientId: '', // Votre Client ID Google
    icsUrl: '',   // OU le lien .ics public (plus simple)
  },

  // Outlook
  outlook: {
    enabled: false,
    clientId: '',
    tenantId: 'common',
    icsUrl: '',
  },

  // Calendriers .ics
  icsCalendars: [
    // { url: 'https://...', type: 'enfant', label: 'Emma - Collège' },
    // { url: 'https://...', type: 'reunion', label: 'Réunions' },
  ],

  // Membres de la famille (pour le partage)
  members: [
    { name: 'Papa',  color: '#059669' },
    { name: 'Maman', color: '#7C3AED' },
    { name: 'Emma',  color: '#1F6FEB' },
    { name: 'Tom',   color: '#D97706' },
  ],
};
```

---

## 5. Déploiement

### Option A — Gratuit et rapide : Netlify Drop

1. Aller sur https://app.netlify.com/drop
2. Glisser-déposer le dossier `calendrier-famille/`
3. Votre app est en ligne en 30 secondes avec une URL publique

### Option B — GitHub Pages (gratuit)

```bash
# Dans le dossier calendrier-famille/
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/VOTRE_COMPTE/familycal.git
git push -u origin main
# Puis dans GitHub : Settings → Pages → Source : main
```

### Option C — VPS / Serveur local (pour un accès maison uniquement)

```bash
# Avec Node.js
npx serve .

# Ou avec Python
python3 -m http.server 8080
# Accéder sur http://localhost:8080
```

---

## 6. Partage en famille

Pour que toute la famille accède au même calendrier :

1. **Héberger** l'app (Netlify ou votre serveur)
2. **Partager l'URL** ou l'ajouter à l'écran d'accueil (la PWA propose automatiquement l'installation sur mobile)
3. Pour synchroniser les événements ajoutés : connecter un **backend simple** (Supabase gratuit) ou utiliser **Google Calendar comme source commune**

---

*Pour toute question, les fichiers sources sont commentés et modulaires.*
