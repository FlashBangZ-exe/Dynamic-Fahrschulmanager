// ==UserScript==
// @name         Fahrschule Dynamic – Prüf-Center (Theorie, Praxis/Pflichtfahrten, Blitzsuche & GSheet-Notizen)
// @namespace    https://portal.fahrschulmanager.de/
// @version      5.8.2
// @description  Vollständige Theorie- & Pflichtfahrten-Analyse (inkl. Optionen-Menü mit Potsdam-Schalter & Dark Mode auf Hauptseite & Prüf-Center, Fahrschulwechsler ABN-Prüfung, B197, Sonderfahrten), Blitzsuche, Kartei & Google-Sheets-Notizen
// @match        https://portal.fahrschulmanager.de/*
// @updateURL    https://raw.githubusercontent.com/FlashBangZ-exe/Dynamic-Fahrschulmanager/main/FSM-Hijack.user.js
// @downloadURL  https://raw.githubusercontent.com/FlashBangZ-exe/Dynamic-Fahrschulmanager/main/FSM-Hijack.user.js
// @run-at       document-start
// @grant        GM_xmlhttpRequest
// @grant        unsafeWindow
// @connect      script.google.com
// @connect      script.googleusercontent.com
// @connect      i.imgur.com
// @connect      imgur.com
// ==/UserScript==

(function () {
  'use strict';

  const BASE_ORIGIN = 'https://portal.fahrschulmanager.de';
  const CUSTOM_LOGO_URL = 'https://i.imgur.com/Y17StTb.png';
  const CUSTOM_LOGO_CACHE_KEY = 'fsm_dyn_custom_logo_dataurl';
  const API_HEADERS = { 'Accept': 'application/json', 'X-CSRF': '1' };
  const pageWin = typeof unsafeWindow !== 'undefined' ? unsafeWindow : window;

  // Optional: Hier kannst du deine Google Apps Script Web-App URL ("https://script.google.com/macros/s/.../exec")
  // fest eintragen – oder später einfach im Kartei-Fenster bei Punkt 5 auf "⚙️ Sheet-URL" klicken!
  const DEFAULT_GSCRIPT_URL = 'https://script.google.com/macros/s/AKfycbxipfwTClq-xrxSjxzs5sTiYaH--syRgB0jo9_Uo78utq_CzOHOTFHDlAYqi6XnO7rmGg/exec';

  let controlCenterActive = false;

  function escapeHtml(val) {
    return String(val ?? '')
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;');
  }

  function isolateKeys(el) {
    if (!el) return;
    ['keydown', 'keyup', 'keypress'].forEach(evt => {
      el.addEventListener(evt, (e) => e.stopPropagation());
    });
  }

  // Umgeht die CSP & den Angular Service Worker (ngsw-worker.js) für externe Aufrufe (Google Sheets / Imgur)
  function externalRequestJson(url, { method = 'GET', headers = {}, body = null, timeout = 12000 } = {}) {
    return new Promise((resolve, reject) => {
      if (typeof GM_xmlhttpRequest === 'function') {
        GM_xmlhttpRequest({
          method,
          url,
          headers,
          data: body,
          timeout,
          onload: (res) => {
            try {
              const parsed = JSON.parse(res.responseText || '{}');
              resolve(parsed);
            } catch (err) {
              reject(new Error('Ungültige JSON-Antwort vom Google Sheet.'));
            }
          },
          onerror: () => reject(new Error('Netzwerkfehler beim Verbinden mit Google Sheet.')),
          ontimeout: () => reject(new Error('Zeitüberschreitung (Timeout) bei Google Sheet.'))
        });
      } else {
        const ctrl = new AbortController();
        const timer = setTimeout(() => ctrl.abort(), timeout);
        fetch(url, {
          method,
          redirect: 'follow',
          headers,
          body,
          signal: ctrl.signal
        })
          .then(r => r.json())
          .then(resolve)
          .catch(reject)
          .finally(() => clearTimeout(timer));
      }
    });
  }

  // Schützt die virtuelle Route /check vor dem Überschreiben durch den Angular 20 Router
  const initialIsCheckRoute = location.pathname.startsWith('/check');
  const initialCheckUrl = location.href;

  // Dark Mode auf der Hauptseite sofort vor dem ersten Rendern aktivieren (mit Null-Schutz für document-start)
  try {
    if (!initialIsCheckRoute && localStorage.getItem('fsm_dynamic_theme') === 'dark') {
      if (document.documentElement) {
        document.documentElement.classList.add('dyn-portal-dark');
      } else {
        document.addEventListener('DOMContentLoaded', () => {
          document.documentElement.classList.add('dyn-portal-dark');
        }, { once: true });
      }
    }
  } catch (e) {}

  // ============================================================================
  // 0. CUSTOM LOGO MANAGER & GOOGLE SHEETS NOTIZEN-MANAGER
  // ============================================================================
  const CustomLogoManager = {
    dataUrl: CUSTOM_LOGO_URL,

    init() {
      try {
        const cached = localStorage.getItem(CUSTOM_LOGO_CACHE_KEY);
        if (cached && cached.startsWith('data:image/')) {
          this.dataUrl = cached;
          return;
        }
      } catch (e) {}
      this.dataUrl = CUSTOM_LOGO_URL;
      this.fetchAndCache();
    },

    getSrc() {
      return this.dataUrl || CUSTOM_LOGO_URL;
    },

    fetchAndCache() {
      const saveBlob = (blob) => {
        if (!blob || !blob.size) return;
        const reader = new FileReader();
        reader.onloadend = () => {
          const result = String(reader.result || '');
          if (result.startsWith('data:image/')) {
            this.dataUrl = result;
            try {
              localStorage.setItem('fsm_dyn_custom_logo_src', CUSTOM_LOGO_URL);
              localStorage.setItem(CUSTOM_LOGO_CACHE_KEY, result);
            } catch (e) {}
            this.apply();
          }
        };
        reader.readAsDataURL(blob);
      };

      if (typeof GM_xmlhttpRequest === 'function') {
        GM_xmlhttpRequest({
          method: 'GET',
          url: CUSTOM_LOGO_URL,
          responseType: 'blob',
          headers: { 'Accept': 'image/png,image/*,*/*' },
          onload: (res) => {
            if (res.status >= 200 && res.status < 300 && res.response) {
              saveBlob(res.response);
            }
          }
        });
      } else {
        fetch(CUSTOM_LOGO_URL, { referrerPolicy: 'no-referrer' })
          .then(r => (r.ok ? r.blob() : null))
          .then(blob => { if (blob) saveBlob(blob); })
          .catch(() => {});
      }
    },

    apply() {
      const src = this.getSrc();
      if (!src) return;

      // 1. Dynamisches Stylesheet für CSS-basierte Logos aktualisieren
      if (document.head) {
        let logoStyle = document.getElementById('dyn-custom-logo-style');
        if (!logoStyle) {
          logoStyle = document.createElement('style');
          logoStyle.id = 'dyn-custom-logo-style';
          document.head.appendChild(logoStyle);
        }
        logoStyle.textContent = `
          app-navbar img[src*="fsmlogo"],
          nav.main-menu img[src*="fsmlogo"],
          img[src*="fsmlogo-.png"],
          img.dyn-custom-logo-img {
            content: url("${src}") !important;
            object-fit: contain !important;
            max-height: 44px !important;
            max-width: 100% !important;
            width: auto !important;
          }
        `;
      }

      // 2. Alle <img>-Elemente (Hauptseite & /check) direkt aktualisieren
      document.querySelectorAll('app-navbar img[src*="fsmlogo"], nav.main-menu img[src*="fsmlogo"], img[src*="fsmlogo"], img.dyn-custom-logo-img').forEach(img => {
        img.classList.add('dyn-custom-logo-img');
        if (img.src !== src) {
          img.src = src;
          img.removeAttribute('srcset');
        }
        img.style.objectFit = 'contain';
        img.style.maxHeight = '44px';
        img.style.maxWidth = '100%';
        img.style.width = 'auto';
      });

      // 3. Falls Angular das Logo als CSS background-image rendert, ebenfalls ersetzen
      document.querySelectorAll('app-navbar *, nav.main-menu *').forEach(el => {
        if (el.dataset.dynBgLogoApplied === '1') return;
        const bg = getComputedStyle(el).backgroundImage;
        if (bg && bg.includes('fsmlogo')) {
          el.dataset.dynBgLogoApplied = '1';
          el.style.setProperty('background-image', `url("${src}")`, 'important');
          el.style.setProperty('background-size', 'contain', 'important');
          el.style.setProperty('background-repeat', 'no-repeat', 'important');
          el.style.setProperty('background-position', 'center', 'important');
        }
      });
    }
  };

  CustomLogoManager.init();

  const NotesManager = {
    map: new Map(),
    listeners: new Set(),
    lastSync: 0,
    isSyncing: false,

    init() {
      try {
        const raw = localStorage.getItem('fsm_dyn_notes_cache');
        if (raw) {
          const obj = JSON.parse(raw);
          Object.entries(obj).forEach(([k, v]) => this.map.set(String(k), v));
        }
      } catch (e) {}
    },

    saveCache() {
      try {
        const obj = {};
        this.map.forEach((v, k) => { obj[k] = v; });
        localStorage.setItem('fsm_dyn_notes_cache', JSON.stringify(obj));
      } catch (e) {}
    },

    notify() {
      this.listeners.forEach(fn => {
        try { fn(); } catch (e) {}
      });
    },

    getEndpoint() {
      return (localStorage.getItem('fsm_dyn_gscript_url') || DEFAULT_GSCRIPT_URL || '').trim();
    },

    setEndpoint(url) {
      const clean = String(url || '').trim();
      if (clean) localStorage.setItem('fsm_dyn_gscript_url', clean);
      else localStorage.removeItem('fsm_dyn_gscript_url');
    },

    get(studentId) {
      return this.map.get(String(studentId)) || { note: '', priority: 'normal', updatedAt: '', updatedBy: '' };
    },

    badgeHtml(studentId) {
      const entry = this.map.get(String(studentId));
      if (!entry || !entry.note) return '';
      const isWarn = entry.priority === 'warn';
      return `<span class="dyn-note-indicator ${isWarn ? 'warn' : ''}" title="${escapeHtml(entry.note)}">${isWarn ? '⚠️' : '📝'} Notiz</span>`;
    },

    async syncAll(force = false) {
      const url = this.getEndpoint();
      if (!url || !url.startsWith('https://script.google.com')) return;
      if (this.isSyncing) return;
      if (!force && Date.now() - this.lastSync < 20000) return;

      this.isSyncing = true;
      try {
        const sep = url.includes('?') ? '&' : '?';
        const data = await externalRequestJson(`${url}${sep}t=${Date.now()}`, {
          method: 'GET',
          timeout: 10000
        });
        if (data && Array.isArray(data.entries)) {
          this.map.clear();
          data.entries.forEach(e => {
            if (e && e.studentId && e.note) {
              this.map.set(String(e.studentId), e);
            }
          });
          this.lastSync = Date.now();
          this.saveCache();
          this.notify();
        }
      } catch (e) {
        console.warn('[Dynamic Notes] Sync-Fehler:', e);
      } finally {
        this.isSyncing = false;
      }
    },

    async save(student, noteText, priority = 'normal') {
      const cleanNote = String(noteText || '').trim();
      const cleanPrio = priority === 'warn' ? 'warn' : 'normal';
      const author = localStorage.getItem('fsm_dynamic_worker_name') || 'Büro';
      const nowStr = new Date().toLocaleString('de-DE', {
        day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit'
      });

      const localEntry = {
        studentId: String(student.id),
        karteiNr: student._karteiNr || '',
        name: `${student._nachname || ''}, ${student._vorname || ''}`.trim(),
        note: cleanNote,
        priority: cleanPrio,
        updatedBy: author,
        updatedAt: nowStr
      };

      if (cleanNote) this.map.set(String(student.id), localEntry);
      else this.map.delete(String(student.id));
      this.saveCache();
      this.notify();

      const url = this.getEndpoint();
      if (!url || !url.startsWith('https://script.google.com')) {
        return { ...localEntry, localOnly: true };
      }

      const payload = {
        studentId: String(student.id),
        karteiNr: localEntry.karteiNr,
        name: localEntry.name,
        note: cleanNote,
        priority: cleanPrio,
        updatedBy: author
      };

      const data = await externalRequestJson(url, {
        method: 'POST',
        headers: { 'Content-Type': 'text/plain;charset=utf-8' },
        body: JSON.stringify(payload),
        timeout: 12000
      });

      if (!data || !data.ok) {
        throw new Error(data?.error || 'Google Sheet hat das Speichern abgelehnt.');
      }

      const finalEntry = data.entry || localEntry;
      if (cleanNote) this.map.set(String(student.id), finalEntry);
      else this.map.delete(String(student.id));
      this.saveCache();
      this.notify();

      return finalEntry;
    }
  };

  NotesManager.init();

  // ============================================================================
  // 1. GEMEINSAMER SCHÜLER- & KARTEI-SPEICHER (INKL. AUTO-SYNC)
  // ============================================================================
  const StudentStore = {
    list: [],
    karteiMap: new Map(),
    lastFetch: 0,
    isFetching: false,
    includeInactive: false,
    listeners: new Set(),

    buildUrl(includeInactive = false) {
      const onlyActive = includeInactive ? 'false' : 'true';
      const url = new URL(`${BASE_ORIGIN}/bff/api/v3/schueler/suche`);
      const params = {
        name: '', vorname: '', bezeichnung: '', kennzeichen: '', strasse: '', ort: '',
        karteiNr: '', telefon: '', klasse: '',
        fidkostentraeger: '', zusatzfeld1: '', zusatzfeld2: '', zusatzfeld3: '', geburtsdatum: '',
        einfacheSuche: 'true', bankeinzug: '', searchText: '',
        onlyActive,
        'filter.reference': 'Alle', 'filter.label': 'Alle', 'filter.type': 'Alle',
        'filter.typeLabel': 'Alle', 'filter.activeFlag': onlyActive, 'filter.listFlag': 'true',
        'filter.archivFlag': 'false', 'filter.letter': '', 'filter.index': '', 'filter.count': '',
        'customFilter.condition': ''
      };
      Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
      return url.toString();
    },

    normalize(s) {
      const klassenStr = Array.isArray(s.klassen) ? s.klassen.join(', ') : String(s.klassen ?? '');
      const nachname = String(s.nachname ?? s.name ?? '');
      const vorname = String(s.vorname ?? '');
      const karteiNr = String(s.karteiNr ?? s.displayKarteinummer ?? '');
      const id = String(s.id ?? '');
      const fullStr = `${vorname} ${nachname} ${karteiNr} ${id}`.toLowerCase();
      const words = fullStr.split(/[\s,.\-_()/]+/).filter(Boolean);
      return {
        ...s,
        _nachname: nachname,
        _vorname: vorname,
        _karteiNr: karteiNr,
        _klassenStr: klassenStr,
        _klassenLower: klassenStr.toLowerCase(),
        _searchLower: fullStr,
        _words: words
      };
    },

    async refresh(force = false, includeInactive = this.includeInactive) {
      if (this.isFetching) return this.list;
      const age = Date.now() - this.lastFetch;
      if (!force && this.list.length > 0 && age < 15000 && includeInactive === this.includeInactive) {
        return this.list;
      }

      this.isFetching = true;
      this.includeInactive = includeInactive;
      try {
        NotesManager.syncAll(force);
        const res = await fetch(this.buildUrl(includeInactive), {
          headers: API_HEADERS,
          credentials: 'include'
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const raw = await res.json();
        const rows = Array.isArray(raw.rows)
          ? raw.rows.map(r => (r && typeof r === 'object') ? (r.d || r.data || r) : r)
          : [];
        this.list = rows.filter(s => s && s.id).map(s => this.normalize(s));
        this.lastFetch = Date.now();
        this.listeners.forEach(fn => fn(this.list));
      } catch (e) {
        console.warn('[Dynamic Store] Sync-Fehler:', e);
      } finally {
        this.isFetching = false;
      }
      return this.list;
    },

    async fetchKartei(studentId, force = false) {
      if (!force && this.karteiMap.has(studentId)) return this.karteiMap.get(studentId);
      const res = await fetch(`${BASE_ORIGIN}/bff/api/v1/schueler/kartei/${encodeURIComponent(studentId)}`, {
        headers: API_HEADERS,
        credentials: 'include'
      });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const data = await res.json();
      this.karteiMap.set(studentId, data);
      return data;
    },

    search(query, { wordStart = true, limit = 15 } = {}) {
      const tokens = (query || '').toLowerCase().trim().split(/[\s,.\-_()/]+/).filter(Boolean);
      if (tokens.length === 0) return [];
      const matches = [];
      for (let i = 0; i < this.list.length; i++) {
        const s = this.list[i];
        let ok = true;
        for (let t = 0; t < tokens.length; t++) {
          const tok = tokens[t];
          if (wordStart) {
            if (!s._words.some(w => w.startsWith(tok))) { ok = false; break; }
          } else {
            if (!s._searchLower.includes(tok)) { ok = false; break; }
          }
        }
        if (ok) {
          matches.push(s);
          if (matches.length >= limit) break;
        }
      }
      return matches;
    }
  };

  setInterval(() => {
    StudentStore.refresh(true);
    NotesManager.syncAll(true);
  }, 120000);

  document.addEventListener('visibilitychange', () => {
    if (!document.hidden && Date.now() - StudentStore.lastFetch > 30000) {
      StudentStore.refresh(true);
      NotesManager.syncAll(true);
    }
  });

  // ============================================================================
  // 2. STRUKTURIERTER KARTEI-PARSER (INKL. ABTEIL 5: BÜRO-NOTIZ)
  // ============================================================================
  function pickProp(obj, ...candidates) {
    if (!obj || typeof obj !== 'object') return undefined;
    const entries = Object.entries(obj);
    for (const cand of candidates) {
      const normCand = cand.replace(/_/g, '').toLowerCase();
      for (const [k, v] of entries) {
        if (k.replace(/_/g, '').toLowerCase() === normCand && v !== null && v !== undefined && v !== '') {
          return v;
        }
      }
    }
    return undefined;
  }

  function formatGermanDate(val, withAge = false) {
    if (!val) return '—';
    const str = String(val).trim();
    const m = str.match(/^(\d{4})-(\d{2})-(\d{2})/);
    if (!m) return str;
    const formatted = `${m[3]}.${m[2]}.${m[1]}`;
    if (!withAge) return formatted;

    const birth = new Date(Number(m[1]), Number(m[2]) - 1, Number(m[3]));
    const now = new Date();
    let age = now.getFullYear() - birth.getFullYear();
    const mDiff = now.getMonth() - birth.getMonth();
    if (mDiff < 0 || (mDiff === 0 && now.getDate() < birth.getDate())) age--;
    return `${formatted} (${age} J.)`;
  }

  function formatHours(val) {
    const num = Number(val);
    if (!num || Number.isNaN(num)) return '0';
    const rounded = Math.round(num * 10) / 10;
    return String(rounded).replace('.', ',');
  }

  function yesNoBadge(val, warnOnTrue = false) {
    const isTrue = Boolean(val) && val !== 'Nein' && val !== 0;
    if (isTrue) {
      return warnOnTrue
        ? `<span class="dyn-kpill warn">⚠️ Ja</span>`
        : `<span class="dyn-kpill yes">✓ Ja</span>`;
    }
    return `<span class="dyn-kpill no">Nein</span>`;
  }

  function buildKarteiCardsHtml(k, studentSummary = {}) {
    if (!k || typeof k !== 'object' || k.error) {
      return `<div style="padding:14px; color:#b3261e;">Keine Karteidaten empfangen (${k?.error || 'Unbekannt'}).</div>`;
    }

    const studentId = studentSummary.id || pickProp(k, 'id') || '';
    const karteiNr = pickProp(k, 'displayKarteinummer', 'karteiNr') || studentSummary._karteiNr || '—';
    const anrede = pickProp(k, 'anrede') || '';
    const vorname = pickProp(k, 'vorname') || studentSummary._vorname || '';
    const nachname = pickProp(k, 'nachname', 'name') || studentSummary._nachname || '';
    const fullName = [anrede, vorname, nachname].filter(Boolean).join(' ').trim() || '—';

    const gebDatum = formatGermanDate(pickProp(k, 'geburtsdatum'), true);
    const gebOrt = pickProp(k, 'geburtsort') || '—';
    const strasse = pickProp(k, 'strasse') || '—';
    const plz = pickProp(k, 'plz') || '';
    const ort = pickProp(k, 'ort', 'wohnort') || '';
    const plzOrt = [plz, ort].filter(Boolean).join(' ').trim() || '—';

    const tel = pickProp(k, 'telefonMobil', 'mobil', 'handy', 'telefon') || '';
    const email = pickProp(k, 'email', 'mail') || '';

    const sehhilfe = pickProp(k, 'sehhilfe');
    const bf17 = pickProp(k, 'bf17');
    const b78 = pickProp(k, 'b78');
    const b197 = pickProp(k, 'b197');
    const b197Test = pickProp(k, 'b197Testfahrt');

    const theoIst = Number(pickProp(k, 'theoriestunden') ?? 0);
    const theoSoll = Number(pickProp(k, 'pflichttheoriestunden') ?? 12);

    const rueckDatum = formatGermanDate(pickProp(k, 'ruecklauf_datum', 'ruecklaufDatum'));
    const rueckNr = pickProp(k, 'ruecklaufnummer', 'pruefauftrag') || '';
    const sprache = pickProp(k, 'fidsystempruefungssprache') || '';
    const erteilungRaw = pickProp(k, 'fiderteilungsart') || '';
    const erteilung = erteilungRaw === 'E' ? 'Ersterwerb (E)' : (erteilungRaw === 'W' ? 'Erweiterung (W)' : (erteilungRaw === 'U' ? 'Umschreibung (U)' : erteilungRaw));

    const ausbildungen = Array.isArray(k.ausbildungen) ? k.ausbildungen : [];
    const firstAusb = ausbildungen[0] || {};
    const klasseKurz = pickProp(firstAusb, 'klasseAbkuerzung', 'klasseKennung') || studentSummary._klassenStr || 'B';
    const zusatzIst = Number(pickProp(firstAusb, 'theoriestunden') ?? 0);
    const zusatzSoll = Number(pickProp(firstAusb, 'pflichtTheoriestunden') ?? 2);
    const uebungsStd = formatHours(pickProp(firstAusb, 'uebungsfahrten'));
    const ueberland = Number(pickProp(firstAusb, 'ueberlandfahrten') ?? 0);
    const autobahn = Number(pickProp(firstAusb, 'autobahnfahrten') ?? 0);
    const nacht = Number(pickProp(firstAusb, 'nachtfahrten') ?? 0);
    const sonderOk = ueberland >= 5 && autobahn >= 4 && nacht >= 3;

    const noteObj = NotesManager.get(studentId);
    const isImportant = noteObj.priority === 'warn';
    const hasGScript = Boolean(NotesManager.getEndpoint());

    return `
      <div class="dyn-kcard">
        <div class="dyn-kcard-hdr">1. Person & Anschrift</div>
        <div class="dyn-kcard-body">
          <div class="dyn-krow"><span class="dyn-klbl">Karteinummer</span><span class="dyn-kval"><strong>${escapeHtml(karteiNr)}</strong></span></div>
          <div class="dyn-krow"><span class="dyn-klbl">Anrede / Name</span><span class="dyn-kval">${escapeHtml(fullName)}</span></div>
          <div class="dyn-krow"><span class="dyn-klbl">Geburtsdatum</span><span class="dyn-kval">${escapeHtml(gebDatum)}</span></div>
          <div class="dyn-krow"><span class="dyn-klbl">Geburtsort</span><span class="dyn-kval">${escapeHtml(gebOrt)}</span></div>
          <div class="dyn-krow"><span class="dyn-klbl">Straße</span><span class="dyn-kval">${escapeHtml(strasse)}</span></div>
          <div class="dyn-krow"><span class="dyn-klbl">PLZ / Ort</span><span class="dyn-kval">${escapeHtml(plzOrt)}</span></div>
          ${tel ? `<div class="dyn-krow"><span class="dyn-klbl">Telefon / Mobil</span><span class="dyn-kval">${escapeHtml(tel)}</span></div>` : ''}
          ${email ? `<div class="dyn-krow"><span class="dyn-klbl">E-Mail</span><span class="dyn-kval">${escapeHtml(email)}</span></div>` : ''}
          <div class="dyn-krow"><span class="dyn-klbl">Sehhilfe</span><span class="dyn-kval">${yesNoBadge(sehhilfe, true)}</span></div>
        </div>
      </div>

      <div class="dyn-kcard">
        <div class="dyn-kcard-hdr">2. Ausbildungsstand (${escapeHtml(klasseKurz)})</div>
        <div class="dyn-kcard-body">
          <div class="dyn-krow">
            <span class="dyn-klbl">Theoriestunden (Grundstoff)</span>
            <span class="dyn-kval"><span class="dyn-kpill ${theoIst >= theoSoll ? 'yes' : 'warn'}">${theoIst} / ${theoSoll} DS</span></span>
          </div>
          <div class="dyn-krow">
            <span class="dyn-klbl">Theoriestunden (Zusatz ${escapeHtml(klasseKurz)})</span>
            <span class="dyn-kval"><span class="dyn-kpill ${zusatzIst >= zusatzSoll ? 'yes' : 'warn'}">${zusatzIst} / ${zusatzSoll} DS</span></span>
          </div>
          <div class="dyn-krow">
            <span class="dyn-klbl">Übungsfahrten (Basis)</span>
            <span class="dyn-kval"><strong>${uebungsStd} Std.</strong></span>
          </div>
          <div class="dyn-krow">
            <span class="dyn-klbl">Sonderfahrten (ÜL · AB · NF)</span>
            <span class="dyn-kval">
              <span class="dyn-kpill ${sonderOk ? 'yes' : 'no'}">
                ÜL: ${ueberland}/5 · AB: ${autobahn}/4 · NF: ${nacht}/3
              </span>
            </span>
          </div>
        </div>
      </div>

      <div class="dyn-kcard">
        <div class="dyn-kcard-hdr">3. Schlüsselzahlen & Besonderheiten</div>
        <div class="dyn-kcard-body">
          <div class="dyn-krow"><span class="dyn-klbl">BF17 (Begleitetes Fahren)</span><span class="dyn-kval">${yesNoBadge(bf17)}</span></div>
          <div class="dyn-krow"><span class="dyn-klbl">B78 (Automatik-Eintrag)</span><span class="dyn-kval">${yesNoBadge(b78)}</span></div>
          <div class="dyn-krow"><span class="dyn-klbl">B197 (Schaltkompetenz)</span><span class="dyn-kval">${yesNoBadge(b197)}</span></div>
          <div class="dyn-krow"><span class="dyn-klbl">B197 Testfahrt absolviert</span><span class="dyn-kval">${yesNoBadge(b197Test)}</span></div>
        </div>
      </div>

      ${(rueckNr || rueckDatum !== '—' || erteilung || sprache) ? `
      <div class="dyn-kcard">
        <div class="dyn-kcard-hdr">4. Behördenantrag & Prüfung</div>
        <div class="dyn-kcard-body">
          ${erteilung ? `<div class="dyn-krow"><span class="dyn-klbl">Erteilungsart</span><span class="dyn-kval">${escapeHtml(erteilung)}</span></div>` : ''}
          ${rueckDatum !== '—' ? `<div class="dyn-krow"><span class="dyn-klbl">Antrag-Rücklaufdatum</span><span class="dyn-kval"><strong>${escapeHtml(rueckDatum)}</strong></span></div>` : ''}
          ${rueckNr ? `<div class="dyn-krow"><span class="dyn-klbl">Rücklauf-/Prüfauftrags-Nr.</span><span class="dyn-kval"><code>${escapeHtml(rueckNr)}</code></span></div>` : ''}
          ${sprache ? `<div class="dyn-krow"><span class="dyn-klbl">Prüfungssprache</span><span class="dyn-kval">${escapeHtml(sprache)}</span></div>` : ''}
        </div>
      </div>
      ` : ''}

      <!-- 5. INTERNE BÜRO-NOTIZ (GOOGLE SHEETS SYNC) -->
      <div class="dyn-kcard dyn-note-card ${isImportant ? 'important' : ''}" data-student-id="${escapeHtml(studentId)}">
        <div class="dyn-kcard-hdr dyn-note-hdr" style="display:flex; justify-content:space-between; align-items:center; gap:8px;">
          <span>5. Interne Büro-Notiz ${hasGScript ? '(GSheet Sync)' : '(Lokal)'}</span>
          <span class="dyn-note-meta" style="font-size:10.5px; font-weight:400; text-transform:none; opacity:0.92;">
            ${noteObj.updatedAt ? `${escapeHtml(noteObj.updatedBy || 'Büro')} · ${escapeHtml(noteObj.updatedAt)}` : 'Noch kein Eintrag'}
          </span>
        </div>
        <div class="dyn-kcard-body" style="display:flex; flex-direction:column; gap:7px; padding:9px 12px;">
          <div style="display:flex; gap:4px; flex-wrap:wrap; align-items:center;">
            <button type="button" class="dyn-mini-btn dyn-tag-btn" data-tag="📞 Rückruf nötig: ">📞 Rückruf</button>
            <button type="button" class="dyn-mini-btn dyn-tag-btn" data-tag="📄 Unterlagen fehlen: ">📄 Unterlagen</button>
            <button type="button" class="dyn-mini-btn dyn-tag-btn" data-tag="💶 Zahlung offen: ">💶 Zahlung</button>
            <button type="button" class="dyn-mini-btn dyn-tag-btn" data-tag="⏳ Prüfungssperre: ">⏳ Sperre</button>
            <label style="margin-left:auto; font-size:11.5px; display:inline-flex; align-items:center; gap:4px; cursor:pointer; font-weight:700; color:#b45309; user-select:none;">
              <input type="checkbox" class="dyn-note-prio" ${isImportant ? 'checked' : ''}> ⚠️ Wichtig
            </label>
          </div>
          <textarea class="dyn-note-textarea" rows="3" placeholder="Interne Notiz für das Büro-Team eintragen (Strg+Enter zum Speichern)...">${escapeHtml(noteObj.note || '')}</textarea>
          <div style="display:flex; justify-content:space-between; align-items:center; gap:6px; flex-wrap:wrap;">
            <div style="display:flex; align-items:center; gap:6px;">
              <button type="button" class="dyn-mini-btn dyn-note-cfg" title="Google Apps Script Web-App URL für Team-Sync hinterlegen">
                ⚙️ ${hasGScript ? 'Sheet-URL' : 'Sheet-URL verbinden'}
              </button>
              <span class="dyn-note-status" style="font-size:11px; color:#64748b;">
                ${hasGScript ? 'Bereit (Strg+Enter)' : 'Lokal (Keine Sheet-URL)'}
              </span>
            </div>
            <div style="display:flex; gap:5px;">
              ${noteObj.note ? `<button type="button" class="dyn-mini-btn dyn-note-clear" title="Notiz löschen">🗑️</button>` : ''}
              <button type="button" class="dyn-mini-btn primary dyn-note-save">💾 Speichern</button>
            </div>
          </div>
        </div>
      </div>
    `;
  }

  function bindKarteiNoteEvents(containerEl, student) {
    if (!containerEl || !student || !student.id) return;
    const card = containerEl.querySelector('.dyn-note-card');
    if (!card) return;

    const textarea = card.querySelector('.dyn-note-textarea');
    const prioChk = card.querySelector('.dyn-note-prio');
    const saveBtn = card.querySelector('.dyn-note-save');
    const clearBtn = card.querySelector('.dyn-note-clear');
    const cfgBtn = card.querySelector('.dyn-note-cfg');
    const statusEl = card.querySelector('.dyn-note-status');
    const metaEl = card.querySelector('.dyn-note-meta');

    isolateKeys(textarea);

    card.querySelectorAll('.dyn-tag-btn').forEach(btn => {
      btn.onclick = (e) => {
        e.preventDefault();
        const tag = btn.getAttribute('data-tag') || '';
        const cur = textarea.value;
        textarea.value = cur ? `${cur.trimEnd()}\n${tag}` : tag;
        textarea.focus();
      };
    });

    prioChk.onchange = () => {
      card.classList.toggle('important', prioChk.checked);
    };

    async function triggerSave(textToSave = textarea.value, prioToSave = prioChk.checked ? 'warn' : 'normal') {
      saveBtn.disabled = true;
      const origText = saveBtn.textContent;
      saveBtn.textContent = '⏳ Speichere...';
      statusEl.style.color = '#0067B8';
      statusEl.textContent = 'Synchronisiere mit Google Sheet...';

      try {
        const saved = await NotesManager.save(student, textToSave, prioToSave);
        metaEl.textContent = saved.note
          ? `${saved.updatedBy || 'Büro'} · ${saved.updatedAt || 'Gerade eben'}`
          : 'Notiz gelöscht';
        statusEl.style.color = '#166534';
        statusEl.textContent = saved.localOnly
          ? '✓ Lokal gespeichert (Sheet-URL fehlt noch)'
          : '✓ Im Google Sheet gespeichert!';
      } catch (err) {
        statusEl.style.color = '#b3261e';
        statusEl.textContent = `⚠️ Fehler: ${err.message}`;
      } finally {
        saveBtn.disabled = false;
        saveBtn.textContent = origText;
      }
    }

    saveBtn.onclick = (e) => {
      e.preventDefault();
      triggerSave();
    };

    if (clearBtn) {
      clearBtn.onclick = (e) => {
        e.preventDefault();
        textarea.value = '';
        prioChk.checked = false;
        card.classList.remove('important');
        triggerSave('', 'normal');
      };
    }

    textarea.addEventListener('keydown', (e) => {
      if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') {
        e.preventDefault();
        triggerSave();
      }
    });

    cfgBtn.onclick = (e) => {
      e.preventDefault();
      const current = NotesManager.getEndpoint();
      const input = window.prompt(
        'Google Apps Script Web-App URL ("https://script.google.com/macros/s/.../exec") für den Team-Sync eintragen:\n(Leer lassen zum Zurücksetzen auf lokalen Speicher)',
        current
      );
      if (input === null) return;
      NotesManager.setEndpoint(input);
      if (NotesManager.getEndpoint()) {
        statusEl.style.color = '#0067B8';
        statusEl.textContent = 'Lade Notizen aus Google Sheet...';
        NotesManager.syncAll(true).then(() => {
          statusEl.style.color = '#166534';
          statusEl.textContent = '✓ Mit Google Sheet verbunden!';
          cfgBtn.textContent = '⚙️ Sheet-URL';
        });
      } else {
        statusEl.style.color = '#64748b';
        statusEl.textContent = 'Auf lokalen Speicher umgestellt.';
        cfgBtn.textContent = '⚙️ Sheet-URL verbinden';
      }
    };
  }

  // Navigation zu /check ohne Angular-404-Redirect
  function navigateToCheckCenter(targetHref) {
    try {
      const targetUrl = new URL(targetHref, BASE_ORIGIN);
      history.pushState(null, '', targetUrl.pathname + targetUrl.search);
    } catch (e) {}
    startDynamicControlCenter();
  }

  document.addEventListener('click', (e) => {
    const link = e.target.closest && e.target.closest('a[href^="/check"]');
    if (link && !e.ctrlKey && !e.metaKey && !e.shiftKey) {
      e.preventDefault();
      e.stopPropagation();
      navigateToCheckCenter(link.getAttribute('href'));
    }
  }, true);

  function bootScript() {
    if (initialIsCheckRoute || location.pathname.startsWith('/check')) {
      if (location.href !== initialCheckUrl && initialIsCheckRoute) {
        try { history.replaceState(null, '', initialCheckUrl); } catch (e) {}
      }
      startDynamicControlCenter();
      return;
    }
    injectPortalEnhancements();
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', bootScript, { once: true });
  } else {
    bootScript();
  }

  // ============================================================================
  // ROUTE B: ORIGINALE FSM-WEBSITE (SIDEBAR + BLITZSUCHE + SCHNELL-KARTEI RECHTS + DARK MODE)
  // ============================================================================
  function injectPortalEnhancements() {
    if (document.getElementById('dyn-portal-styles')) return;

    function syncPortalTheme() {
      const isDark = localStorage.getItem('fsm_dynamic_theme') === 'dark';
      document.documentElement.classList.toggle('dyn-portal-dark', isDark);
    }

    syncPortalTheme();
    window.addEventListener('storage', (e) => {
      if (e.key === 'fsm_dynamic_theme') syncPortalTheme();
    });

    if (!document.querySelector('link[href*="Material+Icons"]')) {
      const iconFont = document.createElement('link');
      iconFont.rel = 'stylesheet';
      iconFont.href = 'https://fonts.googleapis.com/icon?family=Material+Icons';
      document.head.appendChild(iconFont);
    }

    const style = document.createElement('style');
    style.id = 'dyn-portal-styles';
    style.textContent = `
      .dyn-toolbar-search {
        position: relative;
        display: inline-flex;
        align-items: center;
        margin-left: auto;
        margin-right: 12px;
        font-family: Roboto, "Helvetica Neue", sans-serif;
      }
      .dyn-toolbar-input-wrap {
        display: flex;
        align-items: center;
        background: #f0f8ff;
        border: 1.5px solid #00394d;
        border-radius: 4px;
        height: 32px;
        padding: 0 8px;
        gap: 6px;
        min-width: 280px;
      }
      .dyn-toolbar-input-wrap:focus-within {
        border-color: #0067B8;
        box-shadow: 0 0 0 2px rgba(0, 103, 184, 0.22);
        background: #ffffff;
      }
      .dyn-toolbar-input {
        border: none !important;
        background: transparent !important;
        outline: none !important;
        font-size: 13px !important;
        color: #18181b !important;
        width: 100%;
        height: 100%;
      }
      .dyn-kbd {
        font-size: 10px;
        font-weight: 700;
        color: #00394d;
        background: rgba(0, 57, 77, 0.1);
        padding: 2px 5px;
        border-radius: 3px;
        white-space: nowrap;
      }
      .dyn-dropdown {
        position: absolute;
        top: 36px;
        right: 0;
        width: 560px;
        max-width: 94vw;
        background: #ffffff;
        border: 1px solid #cbd5e1;
        border-radius: 6px;
        box-shadow: 0 12px 28px rgba(0, 57, 77, 0.25);
        z-index: 99999;
        overflow: hidden;
        display: none;
      }
      .dyn-dropdown.open { display: block; }
      .dyn-drop-hdr {
        background: #00394d;
        color: #ffffff;
        padding: 6px 12px;
        font-size: 11.5px;
        display: flex;
        justify-content: space-between;
        align-items: center;
      }
      .dyn-drop-list { max-height: 350px; overflow-y: auto; }
      .dyn-drop-item {
        padding: 8px 12px;
        border-bottom: 1px solid #e2e8f0;
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 10px;
        cursor: pointer;
        color: #18181b;
      }
      .dyn-drop-item:hover { background: #f0f8ff; }
      .dyn-drop-actions { display: flex; gap: 4px; flex-shrink: 0; }
      .dyn-mini-btn {
        font-size: 11px;
        font-weight: 600;
        padding: 3px 8px;
        border-radius: 3px;
        border: 1px solid #cbd5e1;
        background: #ffffff;
        color: #00394d;
        text-decoration: none;
        cursor: pointer;
        display: inline-flex;
        align-items: center;
        gap: 3px;
      }
      .dyn-mini-btn:hover { background: #00394d; color: #ffffff; border-color: #00394d; }
      .dyn-mini-btn.primary { background: #00394d; color: #ffffff; border-color: #00394d; }
      .dyn-mini-btn.accent { background: #0067B8; color: #ffffff; border-color: #0067B8; }

      .dyn-note-indicator {
        display: inline-flex;
        align-items: center;
        padding: 1px 6px;
        border-radius: 3px;
        font-size: 10.5px;
        font-weight: 600;
        background: #e0f2fe;
        color: #0369a1;
        border: 1px solid #7dd3fc;
        margin-left: 6px;
      }
      .dyn-note-indicator.warn {
        background: #fef3c7;
        color: #92400e;
        border-color: #fcd34d;
      }

      .dyn-k-backdrop {
        position: fixed;
        inset: 0;
        background: rgba(0, 30, 42, 0.55);
        backdrop-filter: blur(2px);
        z-index: 999998;
        display: none;
        align-items: flex-start;
        justify-content: center;
        padding-top: 9vh;
      }
      .dyn-k-backdrop.open { display: flex; }
      .dyn-k-box {
        width: 100%;
        max-width: 620px;
        background: #ffffff;
        border-radius: 8px;
        border: 1px solid #cbd5e1;
        box-shadow: 0 20px 45px rgba(0, 0, 0, 0.35);
        overflow: hidden;
        font-family: Roboto, "Helvetica Neue", sans-serif;
      }

      .dyn-kartei-drawer {
        position: fixed;
        top: 60px;
        right: 0;
        bottom: 0;
        width: 420px;
        max-width: 92vw;
        background: #f1f5f9;
        border-left: 2px solid #00394d;
        box-shadow: -6px 0 24px rgba(0, 0, 0, 0.22);
        z-index: 99997;
        display: flex;
        flex-direction: column;
        transform: translateX(105%);
        transition: transform 0.18s cubic-bezier(0.16, 1, 0.3, 1);
        font-family: Roboto, "Helvetica Neue", sans-serif;
      }
      .dyn-kartei-drawer.open { transform: translateX(0); }
      .dyn-kd-header {
        background: #00394d;
        color: #ffffff;
        padding: 10px 14px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 10px;
      }
      .dyn-kd-actions {
        background: #ffffff;
        padding: 8px 12px;
        border-bottom: 1px solid #cbd5e1;
        display: flex;
        flex-wrap: wrap;
        gap: 6px;
      }
      .dyn-kd-body {
        flex: 1;
        overflow-y: auto;
        padding: 10px;
        display: flex;
        flex-direction: column;
        gap: 10px;
      }
      .dyn-kcard {
        background: #ffffff;
        border: 1px solid #cbd5e1;
        border-radius: 5px;
        overflow: hidden;
        box-shadow: 0 1px 2px rgba(0,0,0,0.03);
      }
      .dyn-kcard.dyn-note-card.important {
        border-color: #f59e0b;
        box-shadow: 0 0 0 1.5px #f59e0b;
      }
      .dyn-kcard.dyn-note-card.important .dyn-note-hdr {
        background: #b45309;
      }
      .dyn-kcard-hdr {
        background: #00394d;
        color: #ffffff;
        font-size: 12px;
        font-weight: 600;
        letter-spacing: 0.3px;
        text-transform: uppercase;
        padding: 5px 12px;
      }
      .dyn-kcard-body { padding: 6px 12px; }
      .dyn-krow {
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 10px;
        padding: 5px 0;
        border-bottom: 1px solid #f1f5f9;
        font-size: 13px;
      }
      .dyn-krow:last-child { border-bottom: none; }
      .dyn-klbl { font-weight: 500; color: #475569; flex: 1.1; }
      .dyn-kval { color: #0f172a; flex: 1.3; text-align: right; word-break: break-word; }

      .dyn-note-textarea {
        width: 100%;
        resize: vertical;
        min-height: 62px;
        padding: 6px 8px;
        border: 1px solid #cbd5e1;
        border-radius: 4px;
        font-family: inherit;
        font-size: 12.5px;
        color: #0f172a;
        background: #f8fafc;
        box-sizing: border-box;
      }
      .dyn-note-textarea:focus {
        outline: none;
        border-color: #0067B8;
        background: #ffffff;
        box-shadow: 0 0 0 2px rgba(0, 103, 184, 0.16);
      }

      /* Eigenes Logo oben links auf der Hauptseite (ersetzt fsmlogo-.png) */
      app-navbar img[src*="fsmlogo"],
      nav.main-menu img[src*="fsmlogo"],
      img[src*="fsmlogo"] {
        content: url("https://i.imgur.com/Y17StTb.png") !important;
        object-fit: contain !important;
        max-height: 44px !important;
        max-width: 100% !important;
        width: auto !important;
      }

      /* --- MAIN SITE & DYNAMIC OVERLAYS DARK MODE --- */
      html.dyn-portal-dark,
      html.dyn-portal-dark body,
      html.dyn-portal-dark mat-sidenav-container,
      html.dyn-portal-dark mat-drawer-content,
      html.dyn-portal-dark .mat-drawer-content,
      html.dyn-portal-dark .mat-sidenav-content,
      html.dyn-portal-dark main {
        background-color: #090d10 !important;
        color: #f1f5f9 !important;
      }

      html.dyn-portal-dark mat-card,
      html.dyn-portal-dark .mat-mdc-card,
      html.dyn-portal-dark .mat-card,
      html.dyn-portal-dark .mat-expansion-panel,
      html.dyn-portal-dark .cdk-overlay-pane .mat-mdc-dialog-surface,
      html.dyn-portal-dark .cdk-overlay-pane .mat-mdc-menu-panel,
      html.dyn-portal-dark .cdk-overlay-pane .mat-mdc-select-panel,
      html.dyn-portal-dark .cdk-overlay-pane .mat-mdc-autocomplete-panel,
      html.dyn-portal-dark .cdk-overlay-pane .mat-datepicker-content {
        background-color: #11171d !important;
        color: #f1f5f9 !important;
        border-color: #26323d !important;
      }

      html.dyn-portal-dark .mat-expansion-panel-header-title,
      html.dyn-portal-dark .mat-expansion-panel-header-description,
      html.dyn-portal-dark .mat-mdc-list-item,
      html.dyn-portal-dark .mat-mdc-menu-item,
      html.dyn-portal-dark .mat-mdc-option,
      html.dyn-portal-dark .mat-mdc-tab .mdc-tab__text-label {
        color: #f1f5f9 !important;
      }

      html.dyn-portal-dark mat-toolbar:not(.fsm-topbar),
      html.dyn-portal-dark .mat-toolbar {
        background-color: #0c2631 !important;
        color: #f1f5f9 !important;
        border-bottom: 1px solid #26323d !important;
      }

      html.dyn-portal-dark table,
      html.dyn-portal-dark .mat-mdc-table,
      html.dyn-portal-dark .mat-table,
      html.dyn-portal-dark .mat-mdc-paginator {
        background-color: #11171d !important;
        color: #f1f5f9 !important;
      }

      html.dyn-portal-dark th,
      html.dyn-portal-dark .mat-mdc-header-cell {
        background-color: #172028 !important;
        color: #94a3b8 !important;
        border-bottom-color: #26323d !important;
      }

      html.dyn-portal-dark td,
      html.dyn-portal-dark .mat-mdc-cell {
        color: #f1f5f9 !important;
        border-bottom-color: rgba(255, 255, 255, 0.08) !important;
      }

      html.dyn-portal-dark tr:nth-child(odd) td {
        background-color: rgba(255, 255, 255, 0.02);
      }

      html.dyn-portal-dark tr:hover td {
        background-color: rgba(46, 128, 197, 0.16) !important;
      }

      html.dyn-portal-dark .mdc-text-field--filled:not(.mdc-text-field--disabled),
      html.dyn-portal-dark .mdc-text-field--outlined:not(.mdc-text-field--disabled) {
        background-color: #172028 !important;
      }

      html.dyn-portal-dark .mat-mdc-input-element,
      html.dyn-portal-dark .mat-mdc-select-value,
      html.dyn-portal-dark .mat-mdc-floating-label,
      html.dyn-portal-dark input:not(.dyn-toolbar-input):not(#dynGlobalInput),
      html.dyn-portal-dark select,
      html.dyn-portal-dark textarea:not(.dyn-note-textarea) {
        background-color: #172028 !important;
        color: #f1f5f9 !important;
        border-color: #26323d !important;
      }

      html.dyn-portal-dark .mat-mdc-outlined-button:not(:disabled),
      html.dyn-portal-dark .mat-mdc-button:not(:disabled) {
        color: #e2e8f0 !important;
        border-color: #26323d !important;
      }

      /* Dynamic Blitzsuche, Dropdown & Schnell-Kartei Drawer im Dark Mode */
      html.dyn-portal-dark .dyn-toolbar-input-wrap {
        background: #172028;
        border-color: #2e4756;
      }
      html.dyn-portal-dark .dyn-toolbar-input-wrap:focus-within {
        background: #11171d;
        border-color: #38bdf8;
      }
      html.dyn-portal-dark .dyn-toolbar-input {
        color: #f1f5f9 !important;
      }
      html.dyn-portal-dark .dyn-toolbar-input-wrap .material-icons {
        color: #38bdf8 !important;
      }
      html.dyn-portal-dark .dyn-kbd {
        background: rgba(56, 189, 248, 0.15);
        color: #38bdf8;
      }

      html.dyn-portal-dark .dyn-dropdown,
      html.dyn-portal-dark .dyn-k-box,
      html.dyn-portal-dark .dyn-kd-actions,
      html.dyn-portal-dark .dyn-kcard {
        background: #11171d;
        border-color: #26323d;
        color: #f1f5f9;
      }
      html.dyn-portal-dark .dyn-kartei-drawer {
        background: #090d10;
        border-left-color: #26323d;
      }
      html.dyn-portal-dark .dyn-drop-item {
        border-bottom-color: #26323d;
        color: #f1f5f9;
      }
      html.dyn-portal-dark .dyn-drop-item:hover {
        background: rgba(46, 128, 197, 0.16);
      }
      html.dyn-portal-dark .dyn-krow {
        border-bottom-color: rgba(255, 255, 255, 0.08);
      }
      html.dyn-portal-dark .dyn-klbl {
        color: #94a3b8;
      }
      html.dyn-portal-dark .dyn-kval {
        color: #f1f5f9;
      }
      html.dyn-portal-dark #dynGlobalStatus {
        background: #172028 !important;
        border-bottom-color: #26323d !important;
        color: #94a3b8 !important;
      }
      html.dyn-portal-dark .dyn-mini-btn {
        background: #172028;
        border-color: #26323d;
        color: #f1f5f9;
      }
      html.dyn-portal-dark .dyn-mini-btn:hover {
        background: #0067B8;
        border-color: #0067B8;
        color: #ffffff;
      }
      html.dyn-portal-dark .dyn-mini-btn.primary {
        background: #0c2631;
        border-color: #2e4756;
        color: #ffffff;
      }
      html.dyn-portal-dark .dyn-note-textarea {
        background: #172028;
        border-color: #26323d;
        color: #f1f5f9;
      }
      html.dyn-portal-dark .dyn-note-textarea:focus {
        background: #11171d;
        border-color: #38bdf8;
      }
      html.dyn-portal-dark .dyn-kpill.no {
        background: #172028;
        color: #94a3b8;
        border-color: #26323d;
      }
    `;
    document.head.appendChild(style);

    const karteiDrawer = document.createElement('aside');
    karteiDrawer.className = 'dyn-kartei-drawer';
    karteiDrawer.innerHTML = `
      <div class="dyn-kd-header">
        <div>
          <div id="dynKdTitle" style="font-size:16px; font-weight:600;">Schüler-Kartei</div>
          <div id="dynKdSub" style="font-size:11.5px; opacity:0.85;">Karteikarte</div>
        </div>
        <button type="button" id="dynKdClose" style="background:rgba(255,255,255,0.15); border:none; color:#fff; width:28px; height:28px; border-radius:4px; cursor:pointer; font-size:15px;">✕</button>
      </div>
      <div class="dyn-kd-actions" id="dynKdActions"></div>
      <div class="dyn-kd-body" id="dynKdBody"></div>
    `;
    document.body.appendChild(karteiDrawer);
    karteiDrawer.querySelector('#dynKdClose').onclick = () => karteiDrawer.classList.remove('open');

    async function openStudentKartei(student) {
      if (!student || !student.id) return;

      const nativeRow = document.querySelector(`tr[data-key="${student.id}"]`);
      if (nativeRow) {
        nativeRow.click();
        nativeRow.scrollIntoView({ behavior: 'smooth', block: 'center' });
      }

      const titleEl = document.getElementById('dynKdTitle');
      const subEl = document.getElementById('dynKdSub');
      const actionsEl = document.getElementById('dynKdActions');
      const bodyEl = document.getElementById('dynKdBody');

      titleEl.textContent = `${student._nachname}, ${student._vorname}`;
      subEl.textContent = `Kartei-Nr.: ${student._karteiNr || '—'} · Klasse: ${student._klassenStr || '—'}`;

      actionsEl.innerHTML = `
        <a class="dyn-mini-btn accent" href="/check?mod=theorie&student=${student.id}">
          <i class="material-icons" style="font-size:14px;">fact_check</i> Theorie-Check
        </a>
        <a class="dyn-mini-btn accent" href="/check?mod=praxis&student=${student.id}">
          <i class="material-icons" style="font-size:14px;">add_road</i> Pflichtfahrten
        </a>
        <a class="dyn-mini-btn primary" href="/schueler/${student.id}/leistungen">
          <i class="material-icons" style="font-size:14px;">list_alt</i> Leistungen
        </a>
        <a class="dyn-mini-btn" href="/schueler/${student.id}" title="Stammdaten bearbeiten">
          <i class="material-icons" style="font-size:14px;">edit</i> Bearbeiten
        </a>
      `;

      bodyEl.innerHTML = `<div style="padding:20px; text-align:center; color:#64748b;">Lade Karteikarte...</div>`;
      karteiDrawer.classList.add('open');

      try {
        const kData = await StudentStore.fetchKartei(student.id);
        bodyEl.innerHTML = buildKarteiCardsHtml(kData, student);
        bindKarteiNoteEvents(bodyEl, student);
      } catch (err) {
        bodyEl.innerHTML = `<div style="padding:16px; color:#b3261e;">Fehler beim Laden der Kartei: ${escapeHtml(err.message)}</div>`;
      }
    }

    const kBackdrop = document.createElement('div');
    kBackdrop.className = 'dyn-k-backdrop';
    kBackdrop.innerHTML = `
      <div class="dyn-k-box">
        <div style="background:#00394d; color:#fff; padding:10px 14px; display:flex; align-items:center; gap:10px;">
          <i class="material-icons" style="font-size:20px; color:#38bdf8;">bolt</i>
          <input id="dynGlobalInput" type="text" placeholder="Dynamic Blitzsuche: Name, Vorname oder Kartei-Nr. eingeben..."
            style="flex:1; background:transparent; border:none; outline:none; color:#fff; font-size:15px;" autocomplete="off">
          <label style="font-size:11px; display:inline-flex; align-items:center; gap:4px; cursor:pointer; color:#e2e8f0;">
            <input type="checkbox" id="dynGlobalInactive"> Inkl. Archiv
          </label>
          <button type="button" id="dynGlobalClose" style="background:none; border:none; color:#fff; cursor:pointer; font-size:16px;">✕</button>
        </div>
        <div id="dynGlobalStatus" style="padding:5px 14px; background:#f8fafc; border-bottom:1px solid #e2e8f0; font-size:11.5px; color:#64748b;">
          Lade Schülerverzeichnis...
        </div>
        <div id="dynGlobalResults" style="max-height:380px; overflow-y:auto;"></div>
      </div>
    `;
    document.body.appendChild(kBackdrop);

    const globalInput = kBackdrop.querySelector('#dynGlobalInput');
    const globalInactive = kBackdrop.querySelector('#dynGlobalInactive');
    const globalStatus = kBackdrop.querySelector('#dynGlobalStatus');
    const globalResults = kBackdrop.querySelector('#dynGlobalResults');

    isolateKeys(globalInput);

    function bindResultListEvents(containerEl, hits, closeContainerFn) {
      containerEl.querySelectorAll('.dyn-drop-item').forEach((row, idx) => {
        const student = hits[idx];
        row.onclick = (e) => {
          if (e.target.closest('a')) return;
          closeContainerFn();
          openStudentKartei(student);
        };
        const btnKartei = row.querySelector('.btn-open-kartei');
        if (btnKartei) {
          btnKartei.onclick = (e) => {
            e.stopPropagation();
            closeContainerFn();
            openStudentKartei(student);
          };
        }
      });
    }

    function renderHitsHtml(hits) {
      return hits.map(s => {
        const noteEntry = NotesManager.get(s.id);
        return `
        <div class="dyn-drop-item">
          <div style="min-width:0;">
            <div style="font-weight:600; font-size:13.5px; display:flex; align-items:center; flex-wrap:wrap;">
              <span>${escapeHtml(s._nachname)}, ${escapeHtml(s._vorname)}</span>
              ${NotesManager.badgeHtml(s.id)}
            </div>
            <div style="font-size:11.5px; color:#64748b;">
              Kartei: <strong>${escapeHtml(s._karteiNr || '—')}</strong> · Klasse: <strong>${escapeHtml(s._klassenStr || '—')}</strong>
              ${noteEntry.note ? ` · <span style="font-style:italic;">„${escapeHtml(noteEntry.note.slice(0, 45))}${noteEntry.note.length > 45 ? '…' : ''}“</span>` : ''}
            </div>
          </div>
          <div class="dyn-drop-actions">
            <button type="button" class="dyn-mini-btn primary btn-open-kartei" title="Karteikarte & Notiz rechts öffnen">
              <i class="material-icons" style="font-size:13px;">contact_page</i> Kartei
            </button>
            <a class="dyn-mini-btn" href="/schueler/${s.id}/leistungen" title="Leistungen & Stunden">Leistungen</a>
            <a class="dyn-mini-btn accent" href="/check?mod=praxis&student=${s.id}" title="Pflichtfahrten & B197 prüfen">Praxis</a>
            <a class="dyn-mini-btn accent" href="/check?mod=theorie&student=${s.id}" title="Theorie prüfen">Theorie</a>
          </div>
        </div>
      `;
      }).join('');
    }

    async function renderGlobalResults() {
      const q = globalInput.value;
      const inc = globalInactive.checked;
      if (StudentStore.list.length === 0 || StudentStore.includeInactive !== inc) {
        globalStatus.textContent = 'Synchronisiere mit Server...';
        await StudentStore.refresh(true, inc);
      }

      let hits = StudentStore.search(q, { wordStart: true, limit: 20 });
      if (hits.length === 0 && q.trim().length >= 2) {
        if (Date.now() - StudentStore.lastFetch > 8000) {
          globalStatus.textContent = 'Suche nach neuen Schülern auf dem Server...';
          await StudentStore.refresh(true, inc);
          hits = StudentStore.search(q, { wordStart: true, limit: 20 });
        }
        if (hits.length === 0) {
          hits = StudentStore.search(q, { wordStart: false, limit: 20 });
        }
      }

      globalStatus.textContent = q.trim()
        ? `${hits.length} Treffer (Klick öffnet Kartei & Notiz rechts · Enter = 1. Treffer)`
        : `Bereit — ${StudentStore.list.length} Schüler im Schnellzugriff (ESC zum Schließen)`;

      if (!q.trim()) {
        globalResults.innerHTML = `<div style="padding:18px; text-align:center; color:#64748b; font-size:13px;">Tippe einen Namen oder eine Kartei-Nr. ein, um sofort die Karteikarte & Büro-Notiz zu öffnen.</div>`;
        return;
      }

      if (hits.length === 0) {
        globalResults.innerHTML = `<div style="padding:18px; text-align:center; color:#b3261e; font-size:13px;">Kein Fahrschüler gefunden.</div>`;
        return;
      }

      globalResults.innerHTML = renderHitsHtml(hits);
      bindResultListEvents(globalResults, hits, () => kBackdrop.classList.remove('open'));
    }

    function openGlobalSearch() {
      kBackdrop.classList.add('open');
      globalInput.focus();
      globalInput.select();
      renderGlobalResults();
    }

    kBackdrop.onclick = (e) => { if (e.target === kBackdrop) kBackdrop.classList.remove('open'); };
    kBackdrop.querySelector('#dynGlobalClose').onclick = () => kBackdrop.classList.remove('open');
    globalInput.oninput = renderGlobalResults;
    globalInactive.onchange = renderGlobalResults;
    globalInput.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') kBackdrop.classList.remove('open');
      if (e.key === 'Enter') {
        const firstBtn = globalResults.querySelector('.btn-open-kartei');
        if (firstBtn) firstBtn.click();
      }
    });

    document.addEventListener('keydown', (e) => {
      if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'k') {
        e.preventDefault();
        openGlobalSearch();
      }
    });

    function injectDomElements() {
      if (controlCenterActive) return;

      const hoverContainer = document.querySelector('app-navbar nav.main-menu div.hover');
      if (hoverContainer && !hoverContainer.querySelector('.dynamic-injected-category')) {
        CustomLogoManager.apply();
        const statistikLink = hoverContainer.querySelector('a[href*="/statistik"]');
        const statistikUl = statistikLink ? statistikLink.closest('ul') : null;
        if (statistikUl) {
          const ngAttr = Array.from(statistikUl.attributes).find(a => a.name.startsWith('_ngcontent-'));
          const applyNg = (el) => { if (ngAttr) el.setAttribute(ngAttr.name, ''); return el; };

          const createItem = (href, icon, label, onClickFn = null) => {
            const li = applyNg(document.createElement('li'));
            const a = applyNg(document.createElement('a'));
            a.href = href;
            if (onClickFn) {
              a.onclick = (e) => { e.preventDefault(); e.stopPropagation(); onClickFn(); };
            }
            const spanIcon = applyNg(document.createElement('span'));
            spanIcon.className = 'icons';
            const i = applyNg(document.createElement('i'));
            i.className = 'menu-icon material-icons md-dark';
            i.style.color = '#38bdf8';
            i.textContent = icon;
            const spanText = applyNg(document.createElement('span'));
            spanText.className = 'nav-text';
            spanText.textContent = label;

            spanIcon.appendChild(i);
            a.appendChild(spanIcon);
            a.appendChild(spanText);
            li.appendChild(a);
            return li;
          };

          const sep = applyNg(document.createElement('span'));
          sep.className = 'separator dynamic-injected-sep';

          const ul = applyNg(document.createElement('ul'));
          ul.className = 'dynamic-injected-category';
          ul.appendChild(createItem('#blitzsuche', 'bolt', 'Blitzsuche (Strg+K)', openGlobalSearch));
          ul.appendChild(createItem('/check?mod=theorie', 'fact_check', 'Theorie-Check'));
          ul.appendChild(createItem('/check?mod=praxis', 'add_road', 'Pflichtfahrten-Check'));
          ul.appendChild(createItem('/check?mod=dokumente', 'assignment_turned_in', 'Kartei & Akten-Check'));

          statistikUl.insertAdjacentElement('afterend', ul);
          statistikUl.insertAdjacentElement('afterend', sep);
        }
      }

      const schuelerToolbar = document.querySelector('app-schueler > mat-toolbar') || document.querySelector('app-schueler mat-toolbar');
      if (schuelerToolbar && !schuelerToolbar.querySelector('.dyn-toolbar-search')) {
        const wrap = document.createElement('div');
        wrap.className = 'dyn-toolbar-search';
        wrap.innerHTML = `
          <div class="dyn-toolbar-input-wrap">
            <i class="material-icons" style="font-size:17px; color:#00394d;">bolt</i>
            <input type="text" class="dyn-toolbar-input" placeholder="Dynamic Blitzsuche (Klick öffnet Kartei)..." autocomplete="off">
            <span class="dyn-kbd">Strg+K</span>
          </div>
          <div class="dyn-dropdown">
            <div class="dyn-drop-hdr">
              <span class="dyn-drop-count">Blitzsuche</span>
              <button type="button" class="dyn-refresh-mini" style="background:none; border:none; color:#38bdf8; font-size:11px; cursor:pointer; font-weight:600;">↻ Aktualisieren</button>
            </div>
            <div class="dyn-drop-list"></div>
          </div>
        `;

        const input = wrap.querySelector('.dyn-toolbar-input');
        const dropdown = wrap.querySelector('.dyn-dropdown');
        const listEl = wrap.querySelector('.dyn-drop-list');
        const countEl = wrap.querySelector('.dyn-drop-count');
        const refreshBtn = wrap.querySelector('.dyn-refresh-mini');

        isolateKeys(input);

        async function updateInlineDropdown() {
          const q = input.value;
          if (!q.trim()) {
            dropdown.classList.remove('open');
            return;
          }
          dropdown.classList.add('open');

          if (StudentStore.list.length === 0) {
            countEl.textContent = 'Lade Schülerliste...';
            await StudentStore.refresh(true);
          }

          let hits = StudentStore.search(q, { wordStart: true, limit: 15 });
          if (hits.length === 0 && Date.now() - StudentStore.lastFetch > 8000) {
            countEl.textContent = 'Prüfe auf neue Schüler...';
            await StudentStore.refresh(true);
            hits = StudentStore.search(q, { wordStart: true, limit: 15 });
          }
          if (hits.length === 0) {
            hits = StudentStore.search(q, { wordStart: false, limit: 15 });
          }

          countEl.textContent = `${hits.length} Treffer — Klick öffnet Kartei & Notiz rechts`;
          if (hits.length === 0) {
            listEl.innerHTML = `<div style="padding:12px; color:#b3261e; font-size:12.5px;">Kein aktiver Schüler gefunden. (Tipp: Strg+K für Archivsuche)</div>`;
            return;
          }

          listEl.innerHTML = renderHitsHtml(hits);
          bindResultListEvents(listEl, hits, () => dropdown.classList.remove('open'));
        }

        input.oninput = updateInlineDropdown;
        input.onfocus = () => {
          if (StudentStore.list.length === 0) StudentStore.refresh(true);
          if (input.value.trim()) updateInlineDropdown();
        };
        input.addEventListener('keydown', (e) => {
          if (e.key === 'Escape') dropdown.classList.remove('open');
          if (e.key === 'Enter') {
            const firstBtn = listEl.querySelector('.btn-open-kartei');
            if (firstBtn) firstBtn.click();
          }
        });
        refreshBtn.onclick = async (e) => {
          e.stopPropagation();
          countEl.textContent = 'Aktualisiere...';
          await StudentStore.refresh(true);
          await NotesManager.syncAll(true);
          updateInlineDropdown();
        };
        document.addEventListener('click', (e) => {
          if (!wrap.contains(e.target)) dropdown.classList.remove('open');
        });

        schuelerToolbar.appendChild(wrap);
        StudentStore.refresh(false);
      }
    }

    const observer = new MutationObserver(() => injectDomElements());
    observer.observe(document.documentElement, { childList: true, subtree: true });
    injectDomElements();
  }

  // ============================================================================
  // ROUTE C: PRÜF-CENTER APP (/check)
  // ============================================================================
  function startDynamicControlCenter() {
    controlCenterActive = true;
    try { pageWin.stop(); } catch (e) {}
    document.documentElement.innerHTML = getAppHtml();
    CustomLogoManager.apply();
    initAppLogic(BASE_ORIGIN);
  }

  function getAppHtml() {
    return `
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Dynamic Tools - Fahrschul-Manager</title>
      <link rel="preconnect" href="https://fonts.googleapis.com">
      <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;700&display=swap">
      <link rel="stylesheet" href="https://fonts.googleapis.com/icon?family=Material+Icons">
      <style>
        :root {
          --fsm-petrol: #00394d;
          --fsm-petrol-hover: #004d66;
          --fsm-petrol-row-odd: rgba(0, 57, 77, 0.045);
          --fsm-petrol-row-hover: rgba(0, 57, 77, 0.14);
          --fsm-petrol-row-sel: rgba(0, 57, 77, 0.22);
          --fsm-filter-bg: #f0f8ff;
          --dynamic-blue: #0067B8;

          --bg: #f5f5f5;
          --surface: #ffffff;
          --surface-muted: #f8fafc;
          --border: #d4d8dd;
          --border-cell: rgba(0, 0, 0, 0.12);
          --text: #18181b;
          --text-secondary: #52525b;
          --text-muted: #707070;

          --ok-bg: #ecfdf5;
          --ok-border: #a7f3d0;
          --ok-text: #065f46;
          --ok-solid: #16a34a;

          --err-bg: #fdecea;
          --err-border: #f5c2c7;
          --err-text: #b3261e;
          --err-solid: #dc2626;

          --warn-bg: #fffbeb;
          --warn-border: #fde68a;
          --warn-text: #92400e;

          --ext-bg: #f5f3ff;
          --ext-border: #ddd6fe;
          --ext-text: #5b21b6;
        }

        html.dark {
          --fsm-petrol: #0c2631;
          --fsm-petrol-hover: #133949;
          --fsm-petrol-row-odd: rgba(255, 255, 255, 0.025);
          --fsm-petrol-row-hover: rgba(46, 128, 197, 0.16);
          --fsm-petrol-row-sel: rgba(46, 128, 197, 0.26);
          --fsm-filter-bg: #182229;

          --bg: #090d10;
          --surface: #11171d;
          --surface-muted: #172028;
          --border: #26323d;
          --border-cell: rgba(255, 255, 255, 0.1);
          --text: #f1f5f9;
          --text-secondary: #94a3b8;
          --text-muted: #64748b;

          --ok-bg: rgba(22, 163, 74, 0.14);
          --ok-border: rgba(22, 163, 74, 0.4);
          --ok-text: #6ee7b7;
          --ok-solid: #059669;

          --err-bg: rgba(220, 38, 38, 0.15);
          --err-border: rgba(220, 38, 38, 0.45);
          --err-text: #fca5a5;
          --err-solid: #dc2626;

          --warn-bg: rgba(217, 119, 6, 0.15);
          --warn-border: rgba(217, 119, 6, 0.4);
          --warn-text: #fcd34d;

          --ext-bg: rgba(124, 58, 237, 0.16);
          --ext-border: rgba(124, 58, 237, 0.45);
          --ext-text: #c4b5fd;
        }

        * { box-sizing: border-box; }
        body {
          font-family: Roboto, "Helvetica Neue", -apple-system, sans-serif;
          background: var(--bg);
          color: var(--text);
          margin: 0;
          padding: 0;
          font-size: 13px;
          line-height: 1.4;
          font-variant-numeric: tabular-nums;
          overflow-x: hidden;
        }

        .fsm-sidebar {
          background: #00394d;
          position: fixed;
          top: 0;
          bottom: 0;
          left: 0;
          width: 90px;
          z-index: 1000;
          display: flex;
          flex-direction: column;
          padding: 5px 15px;
          overflow: hidden;
          transition: width 0.08s linear;
          user-select: none;
        }
        .fsm-sidebar:hover { width: 250px; box-shadow: 4px 0 16px rgba(0,0,0,0.25); }
        .fsm-logo-head {
          display: flex;
          align-items: center;
          gap: 12px;
          height: 52px;
          padding: 4px 4px;
          color: #ffffff;
          text-decoration: none;
          white-space: nowrap;
        }
        .fsm-sep {
          display: inline-block;
          height: 1px;
          width: 100%;
          margin: 10px 0;
          border-top: 1px solid #ffffff;
        }
        .fsm-menu-ul { list-style: none; margin: 4px 0; padding: 0; }
        .fsm-menu-ul li { display: block; width: 220px; margin: 4px 0; }
        .fsm-menu-ul li a {
          display: flex;
          align-items: center;
          color: #ffffff;
          font-size: 14px;
          text-decoration: none;
          padding: 4px 0;
          transition: color 0.1s;
          cursor: pointer;
        }
        .fsm-menu-ul li a:hover { color: #aaaaaa; }
        .fsm-menu-ul li a .icons {
          display: inline-block;
          width: 60px;
          text-align: center;
          flex-shrink: 0;
        }
        .fsm-menu-ul li a .icons i { font-size: 22px; vertical-align: middle; }
        .fsm-menu-ul li a .nav-text {
          white-space: nowrap;
          opacity: 0;
          transition: opacity 0.08s;
        }
        .fsm-sidebar:hover .fsm-menu-ul li a .nav-text { opacity: 1; }
        .fsm-menu-ul li a.active-tool { font-weight: 700; color: #38bdf8; }

        .fsm-topbar {
          position: fixed;
          top: 0;
          left: 90px;
          right: 0;
          height: 60px;
          background: var(--fsm-petrol);
          color: #ffffff;
          display: flex;
          align-items: center;
          justify-content: space-between;
          padding: 0 20px;
          z-index: 900;
          border-bottom: 1px solid rgba(255,255,255,0.12);
        }
        .topbar-left { display: flex; align-items: center; gap: 16px; }
        .topbar-title { font-size: 20px; font-weight: 500; margin: 0; display: flex; align-items: center; gap: 10px; }
        .dynamic-pill {
          font-size: 11px;
          font-weight: 700;
          letter-spacing: 0.8px;
          padding: 3px 8px;
          border-radius: 4px;
          background: linear-gradient(135deg, #2E80C5 0%, #3B4998 100%);
          color: #fff;
        }

        .module-tabs { display: flex; align-items: center; gap: 4px; background: rgba(0,0,0,0.22); padding: 3px; border-radius: 6px; }
        .mod-tab {
          border: none;
          background: transparent;
          color: rgba(255,255,255,0.75);
          padding: 6px 12px;
          border-radius: 4px;
          font-family: inherit;
          font-size: 12.5px;
          font-weight: 500;
          cursor: pointer;
          display: inline-flex;
          align-items: center;
          gap: 6px;
        }
        .mod-tab:hover { color: #ffffff; background: rgba(255,255,255,0.08); }
        .mod-tab.active { background: #ffffff; color: #00394d; font-weight: 600; }
        html.dark .mod-tab.active { background: #2E80C5; color: #ffffff; }

        .topbar-right { display: flex; align-items: center; gap: 12px; }
        .v-sep { width: 1px; height: 28px; background: rgba(255,255,255,0.3); }
        .user-display { display: grid; font-size: 12px; line-height: 14px; text-align: left; color: #ffffff; }

        .mode-seg {
          display: inline-flex;
          background: rgba(0,0,0,0.25);
          padding: 2px;
          border-radius: 5px;
          border: 1px solid rgba(255,255,255,0.15);
        }
        .mode-seg button {
          border: none;
          background: transparent;
          color: rgba(255,255,255,0.75);
          padding: 4px 10px;
          border-radius: 4px;
          font-size: 12px;
          font-weight: 500;
          cursor: pointer;
          display: inline-flex;
          align-items: center;
          gap: 4px;
        }
        .mode-seg button.active { background: #ffffff; color: #00394d; font-weight: 600; }
        html.dark .mode-seg button.active { background: #2E80C5; color: #ffffff; }

        /* OPTIONS TRIGGER BUTTON & POPUP MENU */
        .options-trigger-btn {
          background: rgba(255, 255, 255, 0.12);
          border: 1px solid rgba(255, 255, 255, 0.25);
          color: #ffffff;
          height: 34px;
          padding: 0 12px;
          border-radius: 6px;
          font-family: inherit;
          font-size: 12.5px;
          font-weight: 500;
          display: inline-flex;
          align-items: center;
          gap: 6px;
          cursor: pointer;
          transition: background 0.14s, border-color 0.14s;
        }
        .options-trigger-btn:hover,
        .options-trigger-btn.open {
          background: rgba(255, 255, 255, 0.22);
          border-color: rgba(255, 255, 255, 0.45);
        }
        .topbar-potsdam-pill {
          background: #7c3aed;
          color: #ffffff;
          font-size: 10.5px;
          font-weight: 700;
          padding: 2px 6px;
          border-radius: 4px;
          letter-spacing: 0.2px;
          box-shadow: 0 1px 3px rgba(0,0,0,0.25);
        }

        .options-backdrop {
          position: fixed;
          inset: 0;
          background: rgba(0, 20, 30, 0.35);
          backdrop-filter: blur(1px);
          z-index: 1998;
          display: none;
        }
        .options-backdrop.open { display: block; }

        .options-popover {
          position: fixed;
          top: 66px;
          right: 20px;
          width: 390px;
          max-width: calc(100vw - 32px);
          background: var(--surface);
          color: var(--text);
          border: 1px solid var(--border);
          border-radius: 10px;
          box-shadow: 0 18px 42px rgba(0, 25, 38, 0.32);
          z-index: 1999;
          display: none;
          overflow: hidden;
          animation: optPopIn 0.14s cubic-bezier(0.16, 1, 0.3, 1);
        }
        .options-popover.open { display: block; }
        @keyframes optPopIn {
          from { opacity: 0; transform: translateY(-6px) scale(0.98); }
          to { opacity: 1; transform: translateY(0) scale(1); }
        }
        .opt-hdr {
          background: var(--fsm-petrol);
          color: #ffffff;
          padding: 11px 15px;
          display: flex;
          align-items: center;
          justify-content: space-between;
          gap: 10px;
        }
        .opt-hdr-title {
          display: flex;
          align-items: center;
          gap: 8px;
          font-size: 14px;
          font-weight: 600;
        }
        .opt-close-btn {
          background: rgba(255,255,255,0.14);
          border: none;
          color: #ffffff;
          width: 26px;
          height: 26px;
          border-radius: 5px;
          cursor: pointer;
          display: inline-flex;
          align-items: center;
          justify-content: center;
          font-size: 14px;
        }
        .opt-close-btn:hover { background: rgba(255,255,255,0.26); }
        .opt-body {
          padding: 14px;
          display: flex;
          flex-direction: column;
          gap: 12px;
        }
        .opt-section-lbl {
          font-size: 10.5px;
          font-weight: 700;
          text-transform: uppercase;
          letter-spacing: 0.6px;
          color: var(--text-muted);
          margin-bottom: -6px;
        }
        .opt-card {
          background: var(--surface-muted);
          border: 1px solid var(--border);
          border-radius: 7px;
          padding: 11px 13px;
          display: flex;
          align-items: flex-start;
          justify-content: space-between;
          gap: 12px;
          cursor: pointer;
          transition: border-color 0.15s, background 0.15s;
        }
        .opt-card:hover {
          border-color: var(--dynamic-blue);
        }
        .opt-card.active-potsdam {
          background: var(--ext-bg);
          border-color: var(--ext-border);
        }
        .opt-card-info {
          display: flex;
          flex-direction: column;
          gap: 3px;
          flex: 1;
        }
        .opt-card-title {
          font-size: 13px;
          font-weight: 600;
          color: var(--text);
          display: flex;
          align-items: center;
          gap: 6px;
          flex-wrap: wrap;
        }
        .opt-card-desc {
          font-size: 11.5px;
          color: var(--text-secondary);
          line-height: 1.38;
        }
        .opt-footer {
          padding: 9px 14px;
          background: var(--surface-muted);
          border-top: 1px solid var(--border);
          display: flex;
          align-items: center;
          justify-content: space-between;
          font-size: 11px;
          color: var(--text-muted);
        }

        .fsm-workspace {
          margin-left: 90px;
          margin-top: 60px;
          padding: 14px 24px 36px;
          display: grid;
          gap: 14px;
        }

        .fsm-card {
          background: var(--surface);
          border: 1px solid var(--border);
          border-radius: 4px;
          box-shadow: 0 1px 2px rgba(0,0,0,0.04);
          overflow: hidden;
        }
        .fsm-card-header {
          background: var(--fsm-petrol);
          color: #ffffff;
          padding: 7px 14px;
          font-size: 14px;
          font-weight: 500;
          display: flex;
          align-items: center;
          justify-content: space-between;
          gap: 12px;
        }

        .fsm-toolbar {
          padding: 10px 14px;
          background: var(--surface);
          border-bottom: 1px solid var(--border);
          display: flex;
          flex-wrap: wrap;
          gap: 10px;
          align-items: flex-end;
        }
        .field { display: flex; flex-direction: column; gap: 4px; }
        .field-label {
          font-size: 11px;
          font-weight: 500;
          color: var(--text-muted);
          display: flex;
          align-items: center;
          gap: 4px;
        }
        .req-badge {
          font-size: 10px;
          background: var(--fsm-petrol);
          color: #fff;
          padding: 1px 5px;
          border-radius: 3px;
          font-weight: 600;
        }

        input:not([type="checkbox"]), select {
          height: 32px;
          padding: 0 10px;
          border-radius: 4px;
          border: 1px solid var(--border);
          background: var(--surface);
          color: var(--text);
          font-family: inherit;
          font-size: 13px;
        }
        input.fsm-filter-input {
          background: var(--fsm-filter-bg);
          border-color: #b0c4de;
        }
        html.dark input.fsm-filter-input { border-color: #2e4756; }
        input:not([type="checkbox"]):focus, select:focus {
          outline: none;
          border-color: var(--dynamic-blue);
          box-shadow: 0 0 0 2px rgba(0, 103, 184, 0.18);
        }
        input.input-invalid {
          border-color: var(--err-solid) !important;
          background: var(--err-bg) !important;
        }

        .alphabet-bar {
          display: flex;
          flex-wrap: wrap;
          gap: 2px;
          padding: 5px 14px;
          background: var(--surface-muted);
          border-bottom: 1px solid var(--border);
          align-items: center;
        }
        .char-btn {
          border: 1px solid var(--border);
          background: var(--fsm-petrol);
          color: #f5f5f5;
          font-size: 11px;
          font-weight: 500;
          min-width: 24px;
          height: 22px;
          border-radius: 3px;
          cursor: pointer;
          display: inline-flex;
          align-items: center;
          justify-content: center;
          padding: 0 5px;
        }
        .char-btn.selected {
          background: var(--surface);
          color: var(--fsm-petrol);
          border-color: var(--fsm-petrol);
          font-weight: 700;
        }
        html.dark .char-btn.selected { color: #38bdf8; border-color: #38bdf8; }

        .btn {
          height: 32px;
          padding: 0 12px;
          border-radius: 4px;
          font-family: inherit;
          font-size: 12.5px;
          font-weight: 500;
          cursor: pointer;
          display: inline-flex;
          align-items: center;
          justify-content: center;
          gap: 5px;
          border: 1px solid transparent;
          white-space: nowrap;
          text-decoration: none;
        }
        .btn:disabled { opacity: 0.45; cursor: not-allowed; }
        .btn-petrol { background: var(--fsm-petrol); color: #ffffff; }
        .btn-petrol:hover:not(:disabled) { background: var(--fsm-petrol-hover); }
        html.dark .btn-petrol { background: #0067B8; }
        .btn-outline { background: var(--surface); color: var(--text); border-color: var(--border); }
        .btn-outline:hover:not(:disabled) { background: var(--surface-muted); }
        .btn-danger { background: var(--err-solid); color: #ffffff; }
        .btn-sm { height: 24px; padding: 0 8px; font-size: 11.5px; }

        .dyn-mini-btn {
          font-size: 11px;
          font-weight: 600;
          padding: 3px 8px;
          border-radius: 3px;
          border: 1px solid var(--border);
          background: var(--surface);
          color: var(--text);
          text-decoration: none;
          cursor: pointer;
          display: inline-flex;
          align-items: center;
          gap: 3px;
        }
        .dyn-mini-btn:hover { background: var(--fsm-petrol); color: #ffffff; border-color: var(--fsm-petrol); }
        .dyn-mini-btn.primary { background: var(--fsm-petrol); color: #ffffff; border-color: var(--fsm-petrol); }

        .dyn-note-indicator {
          display: inline-flex;
          align-items: center;
          padding: 1px 6px;
          border-radius: 3px;
          font-size: 10.5px;
          font-weight: 600;
          background: #e0f2fe;
          color: #0369a1;
          border: 1px solid #7dd3fc;
          margin-left: 6px;
        }
        .dyn-note-indicator.warn {
          background: var(--warn-bg);
          color: var(--warn-text);
          border-color: var(--warn-border);
        }

        .expert-only { display: none !important; }
        body.mode-expert .expert-only { display: inline-flex !important; }
        body.mode-expert div.expert-only,
        body.mode-expert th.expert-only,
        body.mode-expert td.expert-only,
        body.mode-expert section.expert-only { display: revert !important; }

        .fsm-table-scroll { max-height: 235px; overflow: auto; }
        table.fsm-grid { width: 100%; border-collapse: separate; border-spacing: 0; font-size: 13px; }
        table.fsm-grid thead th {
          position: sticky;
          top: 0;
          background: var(--surface);
          color: var(--text-muted);
          font-size: 11.5px;
          font-weight: 500;
          height: 27px;
          padding: 4px 8px;
          text-align: left;
          border-bottom: 1px solid var(--border-cell);
          z-index: 5;
          white-space: nowrap;
        }
        table.fsm-grid thead th + th { border-left: 1px solid var(--border-cell); }
        table.fsm-grid tbody tr { height: 26px; }
        table.fsm-grid tbody tr:nth-child(odd) { background-color: var(--fsm-petrol-row-odd); }
        table.fsm-grid tbody tr:hover { background-color: var(--fsm-petrol-row-hover); cursor: pointer; }
        table.fsm-grid tbody tr.selected { background-color: var(--fsm-petrol-row-sel); font-weight: 500; }
        table.fsm-grid tbody tr.row-external { background-color: var(--ext-bg) !important; }
        table.fsm-grid tbody tr.row-violation { background-color: var(--err-bg) !important; color: var(--err-text); font-weight: 600; }
        table.fsm-grid tbody tr.row-sonder { background-color: rgba(0, 103, 184, 0.08); }
        table.fsm-grid tbody td {
          padding: 2px 8px;
          height: 26px;
          vertical-align: middle;
          border-bottom: 1px solid var(--border-cell);
          white-space: nowrap;
        }
        table.fsm-grid tbody td + td { border-left: 1px solid var(--border-cell); }
        table.fsm-grid tfoot td {
          position: sticky;
          bottom: 0;
          background: var(--surface-muted);
          border-top: 2px double var(--border);
          font-weight: 500;
          padding: 4px 8px;
          font-size: 12px;
        }

        .fsm-badge {
          display: inline-flex;
          align-items: center;
          gap: 4px;
          padding: 1px 7px;
          border-radius: 3px;
          font-size: 11px;
          font-weight: 600;
          border: 1px solid transparent;
        }
        .b-yes { background: var(--ok-bg); color: var(--ok-text); border-color: var(--ok-border); }
        .b-no { background: var(--err-bg); color: var(--err-text); border-color: var(--err-border); }
        .b-warn { background: var(--warn-bg); color: var(--warn-text); border-color: var(--warn-border); }
        .b-open { color: var(--text-muted); font-weight: 400; }
        .b-ext { background: var(--ext-bg); color: var(--ext-text); border-color: var(--ext-border); }

        .verdict-box {
          border-radius: 4px;
          padding: 14px 18px;
          margin-bottom: 12px;
          display: flex;
          justify-content: space-between;
          align-items: flex-start;
          gap: 16px;
          flex-wrap: wrap;
          border: 1px solid var(--border);
        }
        .verdict-box.yes { background: var(--ok-bg); border-color: var(--ok-border); border-left: 5px solid var(--ok-solid); }
        .verdict-box.no { background: var(--err-bg); border-color: var(--err-border); border-left: 5px solid var(--err-solid); }
        .verdict-headline {
          display: inline-flex;
          align-items: center;
          gap: 8px;
          padding: 5px 12px;
          border-radius: 4px;
          font-size: 16px;
          font-weight: 700;
          color: #ffffff;
        }
        .verdict-box.yes .verdict-headline { background: var(--ok-solid); }
        .verdict-box.no .verdict-headline { background: var(--err-solid); }
        .verdict-reasons { margin: 8px 0 0; padding-left: 18px; font-size: 12.5px; display: grid; gap: 3px; }
        .verdict-stats {
          display: flex;
          gap: 14px;
          background: var(--surface);
          border: 1px solid var(--border);
          padding: 8px 14px;
          border-radius: 4px;
          flex-wrap: wrap;
        }
        .stat-col { display: flex; flex-direction: column; }
        .stat-lbl { font-size: 10.5px; color: var(--text-muted); }
        .stat-val { font-size: 13.5px; font-weight: 700; color: var(--text); }

        .akta-notice {
          background: var(--warn-bg);
          border: 1px solid var(--warn-border);
          color: var(--warn-text);
          padding: 7px 12px;
          border-radius: 4px;
          font-size: 12px;
          font-weight: 600;
          margin-bottom: 12px;
          display: flex;
          align-items: center;
          gap: 8px;
        }

        .praxis-check-grid {
          display: grid;
          grid-template-columns: repeat(auto-fit, minmax(270px, 1fr));
          gap: 10px;
          margin-bottom: 12px;
        }
        .pcheck-card {
          background: var(--surface-muted);
          border: 1px solid var(--border);
          border-radius: 4px;
          padding: 10px 12px;
        }
        .pcheck-hdr {
          display: flex;
          justify-content: space-between;
          align-items: center;
          font-weight: 600;
          font-size: 12.5px;
          margin-bottom: 6px;
          padding-bottom: 5px;
          border-bottom: 1px solid var(--border-cell);
        }
        .pcheck-item {
          display: flex;
          justify-content: space-between;
          align-items: center;
          font-size: 12px;
          padding: 3px 0;
        }

        .wechsler-strip {
          background: var(--surface-muted);
          border: 1px solid var(--border);
          border-radius: 4px;
          padding: 10px 14px;
          margin-bottom: 12px;
        }
        .switch-wrap { display: inline-flex; align-items: center; gap: 10px; cursor: pointer; user-select: none; }
        .sw-input { display: none; }
        .sw-track { width: 36px; height: 20px; background: #9ca3af; border-radius: 999px; position: relative; transition: background 0.15s; flex-shrink: 0; }
        .sw-thumb { width: 16px; height: 16px; background: #fff; border-radius: 50%; position: absolute; top: 2px; left: 2px; transition: transform 0.15s; box-shadow: 0 1px 2px rgba(0,0,0,0.25); }
        .sw-input:checked + .sw-track { background: var(--fsm-petrol); }
        html.dark .sw-input:checked + .sw-track { background: #0067B8; }
        .sw-input:checked + .sw-track .sw-thumb { transform: translateX(16px); }

        .wechsler-drawer { margin-top: 10px; padding-top: 10px; border-top: 1px solid var(--border); display: none; }
        .wechsler-drawer.open { display: block; }
        .w-add-bar {
          display: flex;
          flex-wrap: wrap;
          gap: 8px;
          align-items: flex-end;
          background: var(--surface);
          border: 1px solid var(--border);
          border-radius: 4px;
          padding: 8px 10px;
          margin-bottom: 10px;
        }
        .w-lesson-table {
          width: 100%;
          border-collapse: collapse;
          background: var(--surface);
          border: 1px solid var(--border);
          font-size: 12px;
        }
        .w-lesson-table th {
          background: var(--surface-muted);
          color: var(--text-muted);
          font-weight: 600;
          text-align: left;
          padding: 5px 8px;
          border-bottom: 1px solid var(--border);
        }
        .w-lesson-table td {
          padding: 4px 8px;
          border-bottom: 1px solid var(--border-cell);
          vertical-align: middle;
        }
        .w-lesson-table tr.w-row-expired {
          background: var(--err-bg);
        }
        .w-lesson-date-edit {
          height: 26px !important;
          padding: 0 6px !important;
          font-size: 12px !important;
        }

        .matrix-row { display: flex; flex-wrap: wrap; gap: 4px; margin-bottom: 12px; }
        .m-pill {
          padding: 2px 8px;
          border-radius: 3px;
          font-size: 11.5px;
          font-weight: 500;
          border: 1px solid var(--border);
          background: var(--surface-muted);
        }
        .m-done { background: var(--ok-bg); color: var(--ok-text); border-color: var(--ok-border); }
        .m-double { background: var(--warn-bg); color: var(--warn-text); border-color: var(--warn-border); }
        .m-ext { background: var(--ext-bg); color: var(--ext-text); border-color: var(--ext-border); }
        .m-miss { background: var(--err-bg); color: var(--err-text); border-color: var(--err-border); }

        .split-bottom { display: grid; grid-template-columns: 1fr; gap: 14px; }
        body.mode-expert .split-bottom { grid-template-columns: 1.35fr 0.65fr; }

        .kartei-grid {
          display: grid;
          grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
          gap: 12px;
        }
        .dyn-kcard {
          background: var(--surface);
          border: 1px solid var(--border);
          border-radius: 5px;
          overflow: hidden;
        }
        .dyn-kcard.dyn-note-card.important {
          border-color: #f59e0b;
          box-shadow: 0 0 0 1.5px #f59e0b;
        }
        .dyn-kcard.dyn-note-card.important .dyn-note-hdr {
          background: #b45309;
        }
        .dyn-kcard-hdr {
          background: var(--fsm-petrol);
          color: #ffffff;
          font-size: 12px;
          font-weight: 600;
          text-transform: uppercase;
          letter-spacing: 0.3px;
          padding: 6px 12px;
        }
        .dyn-kcard-body { padding: 8px 12px; }
        .dyn-krow {
          display: flex;
          justify-content: space-between;
          align-items: center;
          gap: 10px;
          padding: 5px 0;
          border-bottom: 1px solid var(--border-cell);
          font-size: 13px;
        }
        .dyn-krow:last-child { border-bottom: none; }
        .dyn-klbl { font-weight: 500; color: var(--text-secondary); flex: 1.1; }
        .dyn-kval { color: var(--text); flex: 1.3; text-align: right; word-break: break-word; }

        .dyn-note-textarea {
          width: 100%;
          resize: vertical;
          min-height: 68px;
          padding: 6px 8px;
          border: 1px solid var(--border);
          border-radius: 4px;
          font-family: inherit;
          font-size: 12.5px;
          color: var(--text);
          background: var(--surface-muted);
          box-sizing: border-box;
        }
        .dyn-note-textarea:focus {
          outline: none;
          border-color: var(--dynamic-blue);
          background: var(--surface);
        }

        .dyn-kpill {
          display: inline-block;
          padding: 2px 8px;
          border-radius: 4px;
          font-size: 11.5px;
          font-weight: 600;
        }
        .dyn-kpill.yes { background: var(--ok-bg); color: var(--ok-text); border: 1px solid var(--ok-border); }
        .dyn-kpill.no { background: var(--surface-muted); color: var(--text-muted); border: 1px solid var(--border); }
        .dyn-kpill.warn { background: var(--warn-bg); color: var(--warn-text); border: 1px solid var(--warn-border); }

        pre.json-pre {
          background: #0b1014;
          color: #e2e8f0;
          padding: 12px;
          margin: 0;
          max-height: 390px;
          overflow: auto;
          font-size: 11.5px;
          font-family: ui-monospace, Consolas, monospace;
        }

        .modal-bg {
          position: fixed;
          inset: 0;
          background: rgba(0, 25, 35, 0.6);
          display: none;
          align-items: center;
          justify-content: center;
          z-index: 9999;
        }
        .modal-bg.open { display: flex; }
        .modal-box {
          background: var(--surface);
          border: 1px solid var(--border);
          border-radius: 4px;
          width: 100%;
          max-width: 340px;
          overflow: hidden;
          box-shadow: 0 10px 25px rgba(0,0,0,0.3);
        }

        #printSheet { display: none; }
        @media print {
          body { background: #fff !important; color: #000 !important; }
          .fsm-sidebar, .fsm-topbar, .fsm-workspace, .modal-bg, .options-backdrop, .options-popover { display: none !important; }
          #printSheet { display: block !important; padding: 12mm 14mm; font-family: Roboto, Arial, sans-serif; color: #000; }
          .p-hdr { display: flex; justify-content: space-between; border-bottom: 2px solid #00394d; padding-bottom: 8px; margin-bottom: 10px; }
          .p-alert { border: 1.5px solid #b3261e; background: #fdecea; color: #b3261e; padding: 7px 10px; font-weight: 700; font-size: 11.5px; text-align: center; margin-bottom: 10px; }
          .p-ext { border: 1.5px solid #5b21b6; background: #f5f3ff; color: #5b21b6; padding: 7px 10px; font-weight: 700; font-size: 11.5px; text-align: center; margin-bottom: 10px; }
          .p-meta { display: grid; grid-template-columns: 1fr 1fr; gap: 5px 20px; font-size: 11.5px; margin-bottom: 10px; padding: 8px 10px; border: 1px solid #ccc; }
          .p-verdict { padding: 10px 12px; margin-bottom: 10px; border: 1.5px solid #00394d; }
          .p-table { width: 100%; border-collapse: collapse; font-size: 10.5px; margin-top: 6px; }
          .p-table th, .p-table td { border: 1px solid #ccc; padding: 3px 6px; text-align: left; }
          .p-sigs { margin-top: 26px; display: grid; grid-template-columns: 1fr 1fr; gap: 40px; font-size: 11px; }
          .p-sig-line { border-top: 1px solid #000; padding-top: 4px; margin-top: 30px; }
        }
      </style>
    </head>
    <body class="mode-office">
      <!-- SIDEBAR -->
      <nav class="fsm-sidebar">
        <a href="https://portal.fahrschulmanager.de/schueler" class="fsm-logo-head" style="justify-content:center;">
          <img src="https://i.imgur.com/Y17StTb.png" referrerpolicy="no-referrer" alt="Logo" style="max-height:44px; max-width:100%; width:auto; height:auto; object-fit:contain; display:block;">
        </a>

        <span class="fsm-sep"></span>
        <ul class="fsm-menu-ul">
          <li><a href="https://portal.fahrschulmanager.de/schueler"><span class="icons"><i class="material-icons">school</i></span><span class="nav-text">Fahrschüler</span></a></li>
          <li><a href="https://portal.fahrschulmanager.de/personal"><span class="icons"><i class="material-icons">badge</i></span><span class="nav-text">Personal</span></a></li>
          <li><a href="https://portal.fahrschulmanager.de/fahrzeug"><span class="icons"><i class="material-icons">directions_car</i></span><span class="nav-text">Fahrzeuge</span></a></li>
        </ul>

        <span class="fsm-sep"></span>
        <ul class="fsm-menu-ul">
          <li><a href="https://portal.fahrschulmanager.de/terminplaner"><span class="icons"><i class="material-icons">query_builder</i></span><span class="nav-text">Terminplaner</span></a></li>
          <li><a href="https://portal.fahrschulmanager.de/pruefungsplaner"><span class="icons"><i class="material-icons">event</i></span><span class="nav-text">Prüfungsplaner</span></a></li>
          <li><a href="https://portal.fahrschulmanager.de/kursplaner"><span class="icons"><i class="material-icons">groups</i></span><span class="nav-text">Kursplaner</span></a></li>
          <li><a href="https://portal.fahrschulmanager.de/statistik"><span class="icons"><i class="material-icons">bar_chart</i></span><span class="nav-text">Statistik</span></a></li>
        </ul>

        <span class="fsm-sep"></span>
        <ul class="fsm-menu-ul">
          <li><a data-side-mod="theorie" class="active-tool"><span class="icons"><i class="material-icons" style="color:#38bdf8;">fact_check</i></span><span class="nav-text">Theorie-Check</span></a></li>
          <li><a data-side-mod="praxis"><span class="icons"><i class="material-icons" style="color:#38bdf8;">add_road</i></span><span class="nav-text">Pflichtfahrten-Check</span></a></li>
          <li><a data-side-mod="dokumente"><span class="icons"><i class="material-icons" style="color:#38bdf8;">assignment_turned_in</i></span><span class="nav-text">Kartei & Notizen</span></a></li>
        </ul>

        <span class="fsm-sep"></span>
        <ul class="fsm-menu-ul">
          <li><a href="https://portal.fahrschulmanager.de/erfassung"><span class="icons"><i class="material-icons">edit</i></span><span class="nav-text">Erfassung</span></a></li>
          <li><a href="https://portal.fahrschulmanager.de/kontrolle"><span class="icons"><i class="material-icons">playlist_add_check</i></span><span class="nav-text">Kontrolle</span></a></li>
        </ul>

        <span class="fsm-sep"></span>
        <ul class="fsm-menu-ul">
          <li><a href="https://portal.fahrschulmanager.de/fahrenlernen/verwaltung" target="_blank"><span class="icons"><i class="material-icons">speed</i></span><span class="nav-text">FL Verwaltung</span></a></li>
          <li><a href="https://portal.fahrschulmanager.de/fahrenlernen/radar" target="_blank"><span class="icons"><i class="material-icons">track_changes</i></span><span class="nav-text">FL Radar</span></a></li>
        </ul>

        <span class="fsm-sep"></span>
        <ul class="fsm-menu-ul">
          <li><a href="https://portal.fahrschulmanager.de/fuehrerscheinauskunft"><span class="icons"><i class="material-icons">info</i></span><span class="nav-text">Ausbildungsinfos</span></a></li>
        </ul>
      </nav>

      <!-- TOPBAR -->
      <header class="fsm-topbar">
        <div class="topbar-left">
          <a href="https://portal.fahrschulmanager.de/schueler" class="btn btn-outline btn-sm" style="background:rgba(255,255,255,0.14); color:#fff; border-color:rgba(255,255,255,0.3); font-weight:600;">
            <i class="material-icons" style="font-size:15px;">arrow_back</i> Zurück zu Fahrschüler
          </a>
          <h1 class="topbar-title">
            <span>Dynamic Tools</span>
            <span class="dynamic-pill">FAHRSCHULE DYNAMIC</span>
          </h1>

          <div class="module-tabs">
            <button type="button" class="mod-tab active" data-mod="theorie">
              <i class="material-icons" style="font-size:16px;">fact_check</i> 1. Theorie-Check
            </button>
            <button type="button" class="mod-tab" data-mod="praxis">
              <i class="material-icons" style="font-size:16px;">add_road</i> 2. Praxis & Pflichtfahrten
            </button>
            <button type="button" class="mod-tab" data-mod="dokumente">
              <i class="material-icons" style="font-size:16px;">contact_page</i> 3. Kartei & Büro-Notizen
            </button>
          </div>
        </div>

        <div class="topbar-right">
          <div class="mode-seg">
            <button type="button" id="btnModeOffice" class="active">Büro</button>
            <button type="button" id="btnModeExpert">
              <i class="material-icons" style="font-size:13px;">lock</i> Experte
            </button>
          </div>

          <button type="button" id="btnOptionsMenu" class="options-trigger-btn" title="Einstellungen öffnen (Potsdam-Schalter & Dark Mode)">
            <i class="material-icons" style="font-size:17px;">tune</i>
            <span>Optionen</span>
            <span id="topbarPotsdamPill" class="topbar-potsdam-pill" style="display:none;">📍 Potsdam</span>
          </button>

          <div class="v-sep"></div>

          <div style="display:flex; align-items:center; gap:8px;">
            <i class="material-icons" style="font-size:22px;">person</i>
            <div class="user-display">
              <span id="userFirst" style="font-weight:500;">Sükrü</span>
              <span id="userLast">Erol</span>
            </div>
          </div>
        </div>
      </header>

      <!-- POPUP OPTIONS MENU -->
      <div id="optionsBackdrop" class="options-backdrop"></div>
      <div id="optionsPopover" class="options-popover" role="dialog" aria-label="Dynamic Optionen">
        <div class="opt-hdr">
          <div class="opt-hdr-title">
            <i class="material-icons" style="font-size:18px; color:#38bdf8;">tune</i>
            <span>Optionen & Filial-Einstellungen</span>
          </div>
          <button type="button" id="btnCloseOptions" class="opt-close-btn" title="Schließen (ESC)">✕</button>
        </div>

        <div class="opt-body">
          <div class="opt-section-lbl">Praxis-Checker · Filiale Potsdam</div>
          <label class="opt-card" id="optPotsdamCard" for="optPotsdamSwitch">
            <div class="opt-card-info">
              <div class="opt-card-title">
                <span>📍 Potsdam-Schalter (AB An-/Abfahrt)</span>
                <span id="optPotsdamStatusBadge" class="fsm-badge b-open">Aus</span>
              </div>
              <div class="opt-card-desc">
                Deaktiviert das Zählen & die Pflicht der <strong>2× 20 Min. An- und Abfahrt</strong> zur Autobahnfahrt (Autobahn liegt direkt an der Filiale Potsdam).
              </div>
            </div>
            <div style="padding-top:2px;">
              <input type="checkbox" id="optPotsdamSwitch" class="sw-input">
              <span class="sw-track" style="display:inline-block;"><span class="sw-thumb"></span></span>
            </div>
          </label>

          <div class="opt-section-lbl">Darstellung</div>
          <label class="opt-card" for="optDarkSwitch">
            <div class="opt-card-info">
              <div class="opt-card-title">
                <span>🌙 Dark Mode (Nachtmodus)</span>
                <span id="optDarkStatusBadge" class="fsm-badge b-open">Tag</span>
              </div>
              <div class="opt-card-desc">
                Wechselt zwischen hellem und dunklem Farbschema für Prüf-Center & Hauptseite.
              </div>
            </div>
            <div style="padding-top:2px;">
              <input type="checkbox" id="optDarkSwitch" class="sw-input">
              <span class="sw-track" style="display:inline-block;"><span class="sw-thumb"></span></span>
            </div>
          </label>
        </div>

        <div class="opt-footer">
          <span>Einstellungen werden lokal gespeichert</span>
          <button type="button" id="btnDoneOptions" class="btn btn-petrol btn-sm">Fertig</button>
        </div>
      </div>

      <!-- WORKSPACE -->
      <main class="fsm-workspace">
        <section class="fsm-card">
          <div class="fsm-toolbar">
            <div class="field" style="flex:1; min-width:220px;">
              <span class="field-label">Schnellsuche (Name, Vorname, Kartei-Nr.)</span>
              <input id="qText" type="text" class="fsm-filter-input" placeholder="Name oder Kartei-Nr. eintippen (z. B. Schulz, Nina Sophie oder KWS 7692)..." autocomplete="off">
            </div>

            <div class="field">
              <span class="field-label">Theorie-Modus</span>
              <select id="qRuleMode">
                <option value="ersterwerb" selected>Ersterwerb (12+2)</option>
                <option value="erweiterung">Erweiterung (6+2)</option>
              </select>
            </div>

            <div class="field" style="min-width:210px;">
              <span class="field-label"><span class="req-badge">PFLICHT</span> Geprüft durch (Büro-Kraft)</span>
              <input id="workerName" type="text" placeholder="Vor- und Nachname eintragen" autocomplete="off">
            </div>

            <button id="btnClearFilter" type="button" class="btn btn-outline">Reset</button>
            <button id="btnReloadList" type="button" class="btn btn-outline" title="Schüler & Google-Sheet-Notizen synchronisieren">↻ Sync</button>

            <!-- EXPERT CONTROLS -->
            <div class="field expert-only">
              <span class="field-label">Such-Modus</span>
              <select id="qMatchMode"><option value="word" selected>Wortanfang</option><option value="contains">Enthält</option></select>
            </div>
            <div class="field expert-only">
              <span class="field-label">Klasse</span>
              <input id="qKlasse" type="text" placeholder="B" style="width:55px;">
            </div>
            <div class="field expert-only">
              <span class="field-label">Bestand</span>
              <select id="qActive"><option value="true" selected>Nur Aktive</option><option value="false">Alle</option></select>
            </div>
            <button id="btnBatch" type="button" class="btn btn-petrol expert-only" disabled>Sichtbare prüfen</button>
            <button id="btnStop" type="button" class="btn btn-danger expert-only" style="display:none;">Stopp</button>
          </div>

          <div class="alphabet-bar" id="alphabetBar"></div>

          <div class="fsm-table-scroll">
            <table class="fsm-grid">
              <thead>
                <tr>
                  <th style="width:110px;">Kartei-Nr.</th>
                  <th>Name, Vorname</th>
                  <th style="width:100px;">Klasse(n)</th>
                  <th style="width:165px;">1. Theorie-Check</th>
                  <th style="width:185px;">2. Pflichtfahrten-Check</th>
                  <th style="width:210px; text-align:right;">Aktion</th>
                </tr>
              </thead>
              <tbody id="tbody"></tbody>
              <tfoot>
                <tr>
                  <td colspan="6" style="display:flex; justify-content:space-between; align-items:center;">
                    <span id="status">Lade Fahrschüler...</span>
                    <button id="btnShowAll" type="button" class="btn btn-outline btn-sm" style="display:none;">Alle Treffer laden</button>
                  </td>
                </tr>
              </tfoot>
            </table>
          </div>
        </section>

        <!-- MODULE 1: THEORIE-PRÜFUNGSREIFE -->
        <div id="modTheorie" class="split-bottom">
          <section class="fsm-card" id="evalCard">
            <div class="fsm-card-header">
              <span id="detailTitle">1. Theorie-Prüfungsreife — Bitte oben einen Fahrschüler auswählen</span>
              <button id="btnPrintTheory" type="button" class="btn btn-outline btn-sm" style="display:none; background:#fff; color:#00394d; font-weight:700; border:none; height:26px;">
                <i class="material-icons" style="font-size:15px;">print</i> Theorie-Prüfbericht drucken (Pflicht für Akte)
              </button>
            </div>

            <div style="padding: 14px;">
              <div id="wechslerContainer" class="wechsler-strip" style="display:none;">
                <div style="display:flex; justify-content:space-between; align-items:center; flex-wrap:wrap; gap:10px;">
                  <label class="switch-wrap">
                    <input type="checkbox" id="chkWechsler" class="sw-input">
                    <span class="sw-track"><span class="sw-thumb"></span></span>
                    <span style="font-weight:600;">Fahrschulwechsler (Theoriestunden & ABN einer Vor-Fahrschule anrechnen)</span>
                  </label>
                  <div style="display:flex; gap:6px; align-items:center; flex-wrap:wrap;">
                    <span id="wechslerAbnHeaderBadge" class="fsm-badge" style="display:none;"></span>
                    <span id="wechslerCountBadge" class="fsm-badge b-ext" style="display:none;"></span>
                  </div>
                </div>

                <div id="wechslerPanel" class="wechsler-drawer">
                  <div style="display:flex; gap:10px; flex-wrap:wrap; align-items:flex-end; margin-bottom:10px;">
                    <div class="field" style="flex:1.4; min-width:220px;">
                      <span class="field-label">Name der bisherigen Fahrschule (lt. Ausbildungsbescheinigung)</span>
                      <input type="text" id="wechslerSchoolName" placeholder="z. B. Fahrschule Muster GmbH">
                    </div>
                    <div class="field" style="min-width:175px;">
                      <span class="field-label"><span class="req-badge">ABN</span> Ausstellungsdatum ABN</span>
                      <input type="date" id="wechslerAbnDate">
                    </div>
                    <div id="wechslerAbnStatus" style="padding-bottom:6px; font-size:12px;"></div>
                    <button type="button" id="btnClearWechsler" class="btn btn-outline btn-sm" style="margin-left:auto;">Alle Fremdstunden löschen</button>
                  </div>

                  <div class="w-add-bar">
                    <div class="field" style="min-width:215px;">
                      <span class="field-label">Lektion / Thema (Vor-Fahrschule)</span>
                      <select id="wechslerLessonSelect">
                        <optgroup label="Grundstoff (Pflicht 1–12)">
                          <option value="G1">Lektion 1 (Grundstoff)</option>
                          <option value="G2">Lektion 2 (Grundstoff)</option>
                          <option value="G3">Lektion 3 (Grundstoff)</option>
                          <option value="G4">Lektion 4 (Grundstoff)</option>
                          <option value="G5">Lektion 5 (Grundstoff)</option>
                          <option value="G6">Lektion 6 (Grundstoff)</option>
                          <option value="G7">Lektion 7 (Grundstoff)</option>
                          <option value="G8">Lektion 8 (Grundstoff)</option>
                          <option value="G9">Lektion 9 (Grundstoff)</option>
                          <option value="G10">Lektion 10 (Grundstoff)</option>
                          <option value="G11">Lektion 11 (Grundstoff)</option>
                          <option value="G12">Lektion 12 (Grundstoff)</option>
                        </optgroup>
                        <optgroup label="Klassenspezifischer Zusatzstoff">
                          <option value="B-1">Zusatz B-1 (Klasse B)</option>
                          <option value="B-2">Zusatz B-2 (Klasse B)</option>
                        </optgroup>
                      </select>
                    </div>

                    <div class="field" style="min-width:165px;">
                      <span class="field-label">Unterrichtsdatum (lt. ABN)</span>
                      <input type="date" id="wechslerLessonDate">
                    </div>

                    <button type="button" id="btnAddWechslerLesson" class="btn btn-petrol">
                      <i class="material-icons" style="font-size:16px;">add</i> Stunde hinzufügen
                    </button>

                    <button type="button" id="btnBulkFillGrund" class="btn btn-outline" title="Alle noch fehlenden Grundstoff-Lektionen (1–12) mit dem gewählten Datum (oder ABN-Datum) anlegen">
                      ⚡ L1–L12 mit Datum füllen
                    </button>
                  </div>

                  <div id="wechslerLessonsContainer"></div>
                </div>
              </div>

              <div id="verdictBox">
                <div style="padding:20px; text-align:center; color:var(--text-muted); border:1px dashed var(--border); border-radius:4px;">
                  Wähle oben in der Schülerliste einen Eintrag mit <strong>„Prüfen“</strong> aus, um Theoriestunden & Pflichtfahrten zu kontrollieren.
                </div>
              </div>

              <div id="fileNoticeBox" class="akta-notice" style="display:none;">
                <i class="material-icons" style="font-size:18px;">assignment_late</i>
                <span>Pflichtanweisung: Dieser Prüfbericht muss ausgedruckt und zwingend in der physischen Schülerakte abgeheftet werden.</span>
              </div>

              <div id="chapterMatrix" class="matrix-row"></div>

              <div class="fsm-table-scroll" style="max-height:250px; border:1px solid var(--border);">
                <table class="fsm-grid">
                  <thead>
                    <tr>
                      <th style="width:95px;">Datum</th>
                      <th style="width:110px;">Zeit</th>
                      <th style="width:70px;">Minuten</th>
                      <th style="width:90px;">Klasse</th>
                      <th>Kapitel</th>
                      <th>Fahrlehrer / Herkunft</th>
                      <th>Filiale / Nachweis</th>
                    </tr>
                  </thead>
                  <tbody id="theoryBody">
                    <tr><td colspan="7" style="color:var(--text-muted);">Noch kein Fahrschüler geladen.</td></tr>
                  </tbody>
                </table>
              </div>
            </div>
          </section>

          <section class="fsm-card expert-only">
            <div class="fsm-card-header">
              <span>JSON-Inspektor (API-Rohdaten)</span>
            </div>
            <pre id="jsonView" class="json-pre">// API-Antworten erscheinen hier im Experten-Modus</pre>
          </section>
        </div>

        <!-- MODULE 2: PRAXIS & PFLICHTFAHRTEN ANALYSE-TOOL -->
        <section id="modPraxis" class="fsm-card" style="display:none;">
          <div class="fsm-card-header">
            <span id="praxisTabTitle">2. Praxis & Pflichtfahrten-Analyse — Bitte oben einen Fahrschüler auswählen</span>
            <button id="btnPrintPraxis" type="button" class="btn btn-outline btn-sm" style="display:none; background:#fff; color:#00394d; font-weight:700; border:none; height:26px;">
              <i class="material-icons" style="font-size:15px;">print</i> Praxis-Prüfbericht drucken (Pflicht für Akte)
            </button>
          </div>

          <div style="padding:14px;">
            <div id="praxisConfigBar" class="wechsler-strip" style="display:none;">
              <div style="display:flex; justify-content:space-between; align-items:center; flex-wrap:wrap; gap:14px;">
                <div style="display:flex; align-items:center; gap:14px; flex-wrap:wrap;">
                  <div class="field">
                    <span class="field-label">Ausbildungs-Kategorie (Praxis)</span>
                    <select id="praxisErwerbMode" style="font-weight:600;">
                      <option value="neuerwerber">Neuerwerber / Ersterwerb (Sonderfahrten + mind. 5× /Gf. vor Sonderfahrt)</option>
                      <option value="umschreiber">Umschreiber (Keine Sonderfahrten nötig)</option>
                    </select>
                  </div>

                  <div class="field">
                    <span class="field-label">Schaltkompetenz (B197)</span>
                    <select id="praxisB197Mode" style="font-weight:600;">
                      <option value="yes">Mit B197 (5× 90 Min. + 1× 60 Min. + 15 Min. Testfahrt B197)</option>
                      <option value="no">Ohne B197 (Reine Automatik B78 / Schalter)</option>
                    </select>
                  </div>
                </div>

                <div style="display:flex; align-items:center; gap:10px; flex-wrap:wrap;">
                  <span id="praxisAutoDetectInfo" style="font-size:11.5px; color:var(--text-secondary);"></span>
                  <button type="button" id="praxisPotsdamStatusBtn" class="fsm-badge b-open" style="cursor:pointer;" title="Klicken, um Optionen (Potsdam-Schalter) zu öffnen"></button>
                </div>
              </div>
            </div>

            <div id="praxisVerdictBox">
              <div style="padding:20px; text-align:center; color:var(--text-muted); border:1px dashed var(--border); border-radius:4px;">
                Klicke oben in der Schülerliste bei einem Schüler auf <strong>„Prüfen“</strong>, um Sonderfahrten (225m ÜL, 180m AB + 2×20m An-/Abfahrt, 135m NF), B197-Stunden und die <code>/Gf.</code>-Chronologie auszuwerten.
              </div>
            </div>

            <div id="praxisCheckCards" class="praxis-check-grid" style="display:none;"></div>

            <div class="fsm-table-scroll" style="max-height:310px; border:1px solid var(--border);">
              <table class="fsm-grid">
                <thead>
                  <tr>
                    <th style="width:95px;">Datum</th>
                    <th style="width:105px;">Zeit</th>
                    <th style="width:65px; text-align:right;">Minuten</th>
                    <th style="width:65px;">KFZ</th>
                    <th>Fahrlehrer</th>
                    <th style="width:55px;">Klasse</th>
                    <th>Art</th>
                    <th>Bemerkung</th>
                    <th style="width:85px; text-align:right;">Betrag</th>
                    <th style="width:75px;">Inhalt (/Gf.)</th>
                    <th>Prüf-Hinweis</th>
                  </tr>
                </thead>
                <tbody id="praxisBody">
                  <tr><td colspan="11" style="color:var(--text-muted);">Noch keine Fahrstunden geladen.</td></tr>
                </tbody>
                <tfoot id="praxisFoot" style="display:none;">
                  <tr>
                    <td colspan="2">Summe aller Fahrstunden</td>
                    <td id="praxisSumMin" style="text-align:right; font-weight:700;">0</td>
                    <td colspan="5"></td>
                    <td id="praxisSumBetrag" style="text-align:right; font-weight:700;">0,00 €</td>
                    <td colspan="2"></td>
                  </tr>
                </tfoot>
              </table>
            </div>
          </div>
        </section>

        <!-- MODULE 3: AUFGERÄUMTE SCHÜLER-KARTEI & NOTIZ -->
        <section id="modDokumente" class="fsm-card" style="display:none;">
          <div class="fsm-card-header">
            <span id="karteiTabTitle">3. Schüler-Kartei & Büro-Notizen</span>
            <div id="karteiTabLinks" style="display:flex; gap:6px;"></div>
          </div>
          <div id="karteiTabBody" style="padding:14px;">
            <div style="padding:20px; text-align:center; color:var(--text-muted); border:1px dashed var(--border); border-radius:4px;">
              Klicke oben in der Schülerliste bei einem Schüler auf <strong>„Kartei“</strong> oder <strong>„Prüfen“</strong>, um die übersichtliche Karteikarte und das Google-Sheet-Notizfeld anzuzeigen.
            </div>
          </div>
        </section>
      </main>

      <!-- PASSWORD MODAL ("Ihsan") -->
      <div id="pwModal" class="modal-bg">
        <div class="modal-box">
          <div class="fsm-card-header"><span>Experten-Modus entsperren</span></div>
          <div style="padding:16px;">
            <p style="margin:0 0 12px; font-size:12.5px; color:var(--text-secondary);">
              Bitte Passwort eingeben, um Massenabfragen und API-Rohdaten freizuschalten:
            </p>
            <div class="field" style="margin-bottom:14px;">
              <span class="field-label">Passwort</span>
              <input type="password" id="pwInput" placeholder="Passwort..." autocomplete="off">
              <span id="pwError" style="display:none; font-size:11.5px; color:var(--err-solid); margin-top:3px;">Passwort nicht korrekt.</span>
            </div>
            <div style="display:flex; justify-content:flex-end; gap:8px;">
              <button type="button" id="btnPwCancel" class="btn btn-outline">Abbrechen</button>
              <button type="button" id="btnPwSubmit" class="btn btn-petrol">Freischalten</button>
            </div>
          </div>
        </div>
      </div>

      <!-- A4 PRINT SHEET -->
      <div id="printSheet"></div>
    </body>`;
  }

  function initAppLogic(baseOrigin) {
    let allStudents = [];
    let visibleStudents = [];
    let selectedLetter = '';
    let renderLimit = 150;
    let lastInspected = null;
    const theoryCache = new Map();
    const praxisCache = new Map();
    let abortController = null;

    // 1. Popup Options Menu (Potsdam-Schalter & Dark Mode)
    const rootHtml = document.documentElement;
    const btnOptionsMenu = document.getElementById('btnOptionsMenu');
    const optionsBackdrop = document.getElementById('optionsBackdrop');
    const optionsPopover = document.getElementById('optionsPopover');
    const topbarPotsdamPill = document.getElementById('topbarPotsdamPill');

    const optPotsdamSwitch = document.getElementById('optPotsdamSwitch');
    const optPotsdamCard = document.getElementById('optPotsdamCard');
    const optPotsdamStatusBadge = document.getElementById('optPotsdamStatusBadge');
    const praxisPotsdamStatusBtn = document.getElementById('praxisPotsdamStatusBtn');

    const optDarkSwitch = document.getElementById('optDarkSwitch');
    const optDarkStatusBadge = document.getElementById('optDarkStatusBadge');

    function isPotsdamMode() {
      return localStorage.getItem('fsm_dynamic_potsdam_mode') === 'true';
    }

    function openOptionsMenu() {
      optionsBackdrop.classList.add('open');
      optionsPopover.classList.add('open');
      btnOptionsMenu.classList.add('open');
    }

    function closeOptionsMenu() {
      optionsBackdrop.classList.remove('open');
      optionsPopover.classList.remove('open');
      btnOptionsMenu.classList.remove('open');
    }

    btnOptionsMenu.onclick = () => {
      if (optionsPopover.classList.contains('open')) closeOptionsMenu();
      else openOptionsMenu();
    };
    optionsBackdrop.onclick = closeOptionsMenu;
    document.getElementById('btnCloseOptions').onclick = closeOptionsMenu;
    document.getElementById('btnDoneOptions').onclick = closeOptionsMenu;
    praxisPotsdamStatusBtn.onclick = openOptionsMenu;

    document.addEventListener('keydown', (e) => {
      if (e.key === 'Escape' && optionsPopover.classList.contains('open')) {
        closeOptionsMenu();
      }
    });

    function syncPotsdamUi(enabled) {
      optPotsdamSwitch.checked = enabled;
      optPotsdamCard.classList.toggle('active-potsdam', enabled);
      topbarPotsdamPill.style.display = enabled ? 'inline-block' : 'none';

      if (enabled) {
        optPotsdamStatusBadge.className = 'fsm-badge b-ext';
        optPotsdamStatusBadge.textContent = 'Aktiv (ohne 2×20m)';
        praxisPotsdamStatusBtn.className = 'fsm-badge b-ext';
        praxisPotsdamStatusBtn.textContent = '📍 Potsdam-Modus: ohne AB An-/Abfahrt ⚙';
      } else {
        optPotsdamStatusBadge.className = 'fsm-badge b-open';
        optPotsdamStatusBadge.textContent = 'Aus (Standard)';
        praxisPotsdamStatusBtn.className = 'fsm-badge b-open';
        praxisPotsdamStatusBtn.textContent = 'Standard-Filiale: inkl. 2×20m AB An-/Abfahrt ⚙';
      }
    }

    function setPotsdamMode(enabled) {
      localStorage.setItem('fsm_dynamic_potsdam_mode', enabled ? 'true' : 'false');
      syncPotsdamUi(enabled);

      // Alle gecachten Praxis-Zeilen in der Haupttabelle sofort aktualisieren
      praxisCache.forEach((_, sid) => updateRowBadges(sid));

      if (lastInspected?.praxis) {
        renderPraxisModuleTab(lastInspected.student, lastInspected.praxis);
      }
    }

    syncPotsdamUi(isPotsdamMode());
    optPotsdamSwitch.onchange = () => setPotsdamMode(optPotsdamSwitch.checked);

    function applyTheme(isDark) {
      rootHtml.classList.toggle('dark', isDark);
      optDarkSwitch.checked = isDark;
      if (isDark) {
        optDarkStatusBadge.className = 'fsm-badge b-ext';
        optDarkStatusBadge.textContent = 'Nacht (Aktiv)';
      } else {
        optDarkStatusBadge.className = 'fsm-badge b-open';
        optDarkStatusBadge.textContent = 'Tag';
      }
      localStorage.setItem('fsm_dynamic_theme', isDark ? 'dark' : 'light');
    }

    applyTheme(localStorage.getItem('fsm_dynamic_theme') === 'dark');
    optDarkSwitch.onchange = () => applyTheme(optDarkSwitch.checked);

    // 2. A-Z Alphabet Strip
    const alphabetBar = document.getElementById('alphabetBar');
    const letters = ['Alle', ...'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split('')];
    letters.forEach(ltr => {
      const b = document.createElement('button');
      b.type = 'button';
      b.className = `char-btn ${(ltr === 'Alle' && !selectedLetter) ? 'selected' : ''}`;
      b.textContent = ltr;
      b.onclick = () => {
        selectedLetter = ltr === 'Alle' ? '' : ltr.toLowerCase();
        alphabetBar.querySelectorAll('.char-btn').forEach(el => el.classList.remove('selected'));
        b.classList.add('selected');
        applyFilters(true);
      };
      alphabetBar.appendChild(b);
    });

    // 3. Module Switching
    const modTabs = document.querySelectorAll('.mod-tab');
    const sideLinks = document.querySelectorAll('[data-side-mod]');

    function switchActiveModule(target) {
      if (!['theorie', 'praxis', 'dokumente'].includes(target)) target = 'theorie';
      modTabs.forEach(t => t.classList.toggle('active', t.getAttribute('data-mod') === target));
      sideLinks.forEach(a => a.classList.toggle('active-tool', a.getAttribute('data-side-mod') === target));
      document.getElementById('modTheorie').style.display = target === 'theorie' ? 'grid' : 'none';
      document.getElementById('modPraxis').style.display = target === 'praxis' ? 'block' : 'none';
      document.getElementById('modDokumente').style.display = target === 'dokumente' ? 'block' : 'none';
      try {
        const url = new URL(location.href);
        url.searchParams.set('mod', target);
        history.replaceState(null, '', url.toString());
      } catch (e) {}
    }

    modTabs.forEach(tab => { tab.onclick = () => switchActiveModule(tab.getAttribute('data-mod')); });
    sideLinks.forEach(link => {
      link.onclick = (e) => { e.preventDefault(); switchActiveModule(link.getAttribute('data-side-mod')); };
    });

    const urlParams = new URLSearchParams(location.search);
    const initialMod = urlParams.get('mod');
    const initialStudentId = urlParams.get('student');
    if (initialMod) switchActiveModule(initialMod);

    // 4. Password-Protected Expert Mode ("Ihsan")
    const btnModeOffice = document.getElementById('btnModeOffice');
    const btnModeExpert = document.getElementById('btnModeExpert');
    const pwModal = document.getElementById('pwModal');
    const pwInput = document.getElementById('pwInput');
    const pwError = document.getElementById('pwError');

    btnModeOffice.onclick = () => {
      document.body.className = 'mode-office';
      btnModeOffice.classList.add('active');
      btnModeExpert.classList.remove('active');
    };
    btnModeExpert.onclick = () => {
      if (document.body.classList.contains('mode-expert')) return;
      pwInput.value = '';
      pwError.style.display = 'none';
      pwInput.classList.remove('input-invalid');
      pwModal.classList.add('open');
      setTimeout(() => pwInput.focus(), 50);
    };

    function verifyExpertPassword() {
      if (pwInput.value === 'Ihsan') {
        pwModal.classList.remove('open');
        document.body.className = 'mode-expert';
        btnModeExpert.classList.add('active');
        btnModeOffice.classList.remove('active');
      } else {
        pwError.style.display = 'block';
        pwInput.classList.add('input-invalid');
        pwInput.select();
      }
    }
    document.getElementById('btnPwSubmit').onclick = verifyExpertPassword;
    document.getElementById('btnPwCancel').onclick = () => pwModal.classList.remove('open');
    pwInput.addEventListener('keydown', (e) => {
      e.stopPropagation();
      if (e.key === 'Enter') verifyExpertPassword();
      if (e.key === 'Escape') pwModal.classList.remove('open');
    });

    // 5. Mandatory Worker Name
    const workerInput = document.getElementById('workerName');
    const savedWorker = localStorage.getItem('fsm_dynamic_worker_name') || '';
    if (savedWorker) workerInput.value = savedWorker;

    workerInput.addEventListener('input', (e) => {
      e.stopPropagation();
      const val = workerInput.value.trim();
      localStorage.setItem('fsm_dynamic_worker_name', val);
      if (val) workerInput.classList.remove('input-invalid');
    });

    function requireWorkerName() {
      const val = workerInput.value.trim();
      if (!val) {
        workerInput.classList.add('input-invalid');
        workerInput.focus();
        const status = document.getElementById('status');
        status.style.color = 'var(--err-solid)';
        status.textContent = '⚠️ Bitte zuerst oben rechts im Pflichtfeld „Geprüft durch (Büro-Kraft)“ deinen Namen eintragen!';
        return null;
      }
      return val;
    }

    // 6. Fahrschulwechsler State & 2-Jahres-Fristprüfung (§ 4 Abs. 6 FahrschAusbO)
    const ALL_WECHSLER_KEYS = ['G1','G2','G3','G4','G5','G6','G7','G8','G9','G10','G11','G12','B-1','B-2'];

    function formatWechslerLessonLabel(key) {
      if (key.startsWith('G')) return `Lektion ${key.slice(1)} (Grundstoff)`;
      return `Zusatz ${key} (Klasse B)`;
    }

    function checkTwoYearValidity(isoDate) {
      if (!isoDate || !/^\d{4}-\d{2}-\d{2}$/.test(isoDate)) {
        return { status: 'missing', label: 'Datum fehlt', expiresDe: '—', daysLeft: null, isExpired: false };
      }
      const [y, m, d] = isoDate.split('-').map(Number);
      const expiryDate = new Date(y + 2, m - 1, d);
      const today = new Date();
      today.setHours(0, 0, 0, 0);

      const diffMs = expiryDate.getTime() - today.getTime();
      const daysLeft = Math.ceil(diffMs / (1000 * 60 * 60 * 24));
      const expiresDe = `${String(expiryDate.getDate()).padStart(2, '0')}.${String(expiryDate.getMonth() + 1).padStart(2, '0')}.${expiryDate.getFullYear()}`;

      if (daysLeft < 0) {
        return {
          status: 'expired',
          label: `Abgelaufen seit ${expiresDe} (${Math.abs(daysLeft)} T.)`,
          expiresDe,
          daysLeft,
          isExpired: true
        };
      }
      if (daysLeft <= 60) {
        return {
          status: 'expiring',
          label: `Läuft ab am ${expiresDe} (noch ${daysLeft} T.)`,
          expiresDe,
          daysLeft,
          isExpired: false
        };
      }
      return {
        status: 'valid',
        label: `Gültig bis ${expiresDe}`,
        expiresDe,
        daysLeft,
        isExpired: false
      };
    }

    function getWechslerData(studentId) {
      if (!studentId) return { enabled: false, schoolName: '', abnDate: '', lessons: [] };
      try {
        const raw = localStorage.getItem(`fsm_wechsler_${studentId}`);
        if (raw) {
          const parsed = JSON.parse(raw);
          if (!Array.isArray(parsed.lessons)) {
            const migrated = [];
            if (parsed.counts && typeof parsed.counts === 'object') {
              for (const [k, cntVal] of Object.entries(parsed.counts)) {
                const cnt = Number(cntVal) || 0;
                for (let i = 0; i < cnt; i++) {
                  migrated.push({ id: `mig_${k}_${i}_${Math.random().toString(36).slice(2, 7)}`, key: k, date: '' });
                }
              }
            }
            parsed.lessons = migrated;
            delete parsed.counts;
          }
          return {
            enabled: Boolean(parsed.enabled),
            schoolName: String(parsed.schoolName || ''),
            abnDate: String(parsed.abnDate || ''),
            lessons: parsed.lessons
          };
        }
      } catch (e) {}
      return { enabled: false, schoolName: '', abnDate: '', lessons: [] };
    }

    function saveWechslerData(studentId, data) {
      if (!studentId) return;
      localStorage.setItem(`fsm_wechsler_${studentId}`, JSON.stringify(data));
    }

    function sortWechslerLessons(lessons) {
      return [...lessons].sort((a, b) => {
        const idxA = ALL_WECHSLER_KEYS.indexOf(a.key);
        const idxB = ALL_WECHSLER_KEYS.indexOf(b.key);
        if (idxA !== idxB) return idxA - idxB;
        return String(a.date || '').localeCompare(String(b.date || ''));
      });
    }

    function buildExternalTheoryItems(studentId) {
      const wData = getWechslerData(studentId);
      if (!wData.enabled) return [];
      const schoolLabel = wData.schoolName.trim() || 'Fremdfahrschule';
      const abnInfo = wData.abnDate ? `ABN v. ${formatGermanDate(wData.abnDate)}` : 'ABN (Datum offen)';
      const sorted = sortWechslerLessons(wData.lessons || []);

      return sorted.map(entry => {
        const key = String(entry.key || '');
        const isoDate = String(entry.date || '').trim();
        const validity = checkTwoYearValidity(isoDate);
        const datumStr = isoDate ? formatGermanDate(isoDate) : 'Datum fehlt';
        const zeitStr = isoDate ? 'Nachweis (ABN)' : 'Ohne Datum';
        const filialeStr = `${abnInfo} · ${validity.label}`;

        if (key.startsWith('G')) {
          const chNum = Number(key.slice(1));
          return {
            isoDate,
            datum: datumStr,
            zeit: zeitStr,
            min: 90,
            units: 1,
            klasse: '',
            kapitel: `${chNum} Grundstoff (Anerkennung Fahrschulwechsel)`,
            kapNum: chNum,
            lehrer: schoolLabel,
            filiale: filialeStr,
            geloescht: false,
            isExternal: true,
            validity
          };
        } else {
          const zNum = Number(key.split('-')[1]) || 1;
          return {
            isoDate,
            datum: datumStr,
            zeit: zeitStr,
            min: 90,
            units: 1,
            klasse: 'B',
            kapitel: `${zNum} Zusatzstoff Klasse B (Anerkennung Fahrschulwechsel)`,
            kapNum: zNum,
            lehrer: schoolLabel,
            filiale: filialeStr,
            geloescht: false,
            isExternal: true,
            validity
          };
        }
      });
    }

    const chkWechsler = document.getElementById('chkWechsler');
    const wechslerPanel = document.getElementById('wechslerPanel');
    const wechslerSchoolName = document.getElementById('wechslerSchoolName');
    const wechslerAbnDate = document.getElementById('wechslerAbnDate');
    const wechslerAbnStatus = document.getElementById('wechslerAbnStatus');
    const wechslerAbnHeaderBadge = document.getElementById('wechslerAbnHeaderBadge');
    const wechslerLessonSelect = document.getElementById('wechslerLessonSelect');
    const wechslerLessonDate = document.getElementById('wechslerLessonDate');
    const wechslerLessonsContainer = document.getElementById('wechslerLessonsContainer');
    const wechslerCountBadge = document.getElementById('wechslerCountBadge');

    isolateKeys(wechslerSchoolName);
    isolateKeys(wechslerAbnDate);
    isolateKeys(wechslerLessonDate);

    function syncWechslerUi(studentId) {
      const wData = getWechslerData(studentId);
      chkWechsler.checked = Boolean(wData.enabled);
      wechslerPanel.classList.toggle('open', Boolean(wData.enabled));
      wechslerSchoolName.value = wData.schoolName || '';
      wechslerAbnDate.value = wData.abnDate || '';

      if (wData.abnDate) {
        const abnVal = checkTwoYearValidity(wData.abnDate);
        const badgeCls = abnVal.status === 'expired' ? 'b-no' : (abnVal.status === 'expiring' ? 'b-warn' : 'b-yes');
        const iconPrefix = abnVal.status === 'expired' ? '❌' : (abnVal.status === 'expiring' ? '⚠️' : '✓');
        wechslerAbnStatus.innerHTML = `<span class="fsm-badge ${badgeCls}">${iconPrefix} ABN ${abnVal.label}</span>`;
        if (wData.enabled) {
          wechslerAbnHeaderBadge.style.display = 'inline-flex';
          wechslerAbnHeaderBadge.className = `fsm-badge ${badgeCls}`;
          wechslerAbnHeaderBadge.textContent = `ABN v. ${formatGermanDate(wData.abnDate)}`;
        } else {
          wechslerAbnHeaderBadge.style.display = 'none';
        }
      } else {
        wechslerAbnStatus.innerHTML = `<span style="color:var(--text-muted); font-size:11.5px;">Bitte Ausstellungsdatum eintragen</span>`;
        wechslerAbnHeaderBadge.style.display = 'none';
      }

      const sortedLessons = sortWechslerLessons(wData.lessons || []);
      const totalExt = sortedLessons.length;
      const expiredExt = sortedLessons.filter(l => checkTwoYearValidity(l.date).isExpired).length;

      if (wData.enabled && totalExt > 0) {
        wechslerCountBadge.style.display = 'inline-flex';
        wechslerCountBadge.className = `fsm-badge ${expiredExt > 0 ? 'b-no' : 'b-ext'}`;
        wechslerCountBadge.textContent = expiredExt > 0
          ? `${totalExt} DS Vor-Fahrschule (${expiredExt} abgelaufen!)`
          : `${totalExt} DS von Vor-Fahrschule`;
      } else {
        wechslerCountBadge.style.display = 'none';
      }

      if (sortedLessons.length === 0) {
        wechslerLessonsContainer.innerHTML = `
          <div style="font-size:12px; color:var(--text-muted); padding:6px 2px;">
            Noch keine Theoriestunden der Vor-Fahrschule eingetragen. Wähle oben Lektion & Unterrichtsdatum aus und klicke auf <strong>„+ Stunde hinzufügen“</strong>.
          </div>
        `;
        return;
      }

      wechslerLessonsContainer.innerHTML = `
        <table class="w-lesson-table">
          <thead>
            <tr>
              <th style="width:40px;">Nr.</th>
              <th>Thema / Lektion</th>
              <th style="width:160px;">Unterrichtsdatum</th>
              <th>Fristprüfung (2 Jahre § 4 Abs. 6 FahrschAusbO)</th>
              <th style="width:65px; text-align:right;">Aktion</th>
            </tr>
          </thead>
          <tbody>
            ${sortedLessons.map((entry, idx) => {
              const val = checkTwoYearValidity(entry.date);
              const badgeCls = val.status === 'expired'
                ? 'b-no'
                : (val.status === 'expiring' || val.status === 'missing' ? 'b-warn' : 'b-yes');
              const prefix = val.status === 'expired'
                ? '❌'
                : (val.status === 'expiring' || val.status === 'missing' ? '⚠️' : '✓');
              return `
                <tr class="${val.isExpired ? 'w-row-expired' : ''}">
                  <td>${idx + 1}</td>
                  <td><strong>${escapeHtml(formatWechslerLessonLabel(entry.key))}</strong></td>
                  <td>
                    <input type="date" class="w-lesson-date-edit" data-lesson-id="${escapeHtml(entry.id)}" value="${escapeHtml(entry.date || '')}">
                  </td>
                  <td><span class="fsm-badge ${badgeCls}">${prefix} ${escapeHtml(val.label)}</span></td>
                  <td style="text-align:right;">
                    <button type="button" class="btn btn-outline btn-sm btn-del-wlesson" data-lesson-id="${escapeHtml(entry.id)}" title="Eintrag entfernen">✕</button>
                  </td>
                </tr>
              `;
            }).join('')}
          </tbody>
        </table>
      `;

      wechslerLessonsContainer.querySelectorAll('.w-lesson-date-edit').forEach(inp => {
        isolateKeys(inp);
        inp.onchange = () => {
          const lid = inp.getAttribute('data-lesson-id');
          const curData = getWechslerData(studentId);
          const target = curData.lessons.find(x => x.id === lid);
          if (target) {
            target.date = inp.value;
            saveWechslerData(studentId, curData);
            syncWechslerUi(studentId);
            refreshCurrentEvaluation();
          }
        };
      });

      wechslerLessonsContainer.querySelectorAll('.btn-del-wlesson').forEach(btn => {
        btn.onclick = () => {
          const lid = btn.getAttribute('data-lesson-id');
          const curData = getWechslerData(studentId);
          curData.lessons = curData.lessons.filter(x => x.id !== lid);
          saveWechslerData(studentId, curData);
          syncWechslerUi(studentId);
          refreshCurrentEvaluation();
        };
      });
    }

    function addWechslerLessonEntry() {
      if (!lastInspected?.student?.id) return;
      const id = lastInspected.student.id;
      const wData = getWechslerData(id);
      const selectedKey = wechslerLessonSelect.value;
      const chosenDate = wechslerLessonDate.value || wData.abnDate || '';

      if (!chosenDate) {
        wechslerLessonDate.classList.add('input-invalid');
        wechslerLessonDate.focus();
        setTimeout(() => wechslerLessonDate.classList.remove('input-invalid'), 1800);
        return;
      }

      if (!wechslerLessonDate.value && chosenDate) {
        wechslerLessonDate.value = chosenDate;
      }

      wData.enabled = true;
      wData.lessons.push({
        id: `wl_${Date.now()}_${Math.random().toString(36).slice(2, 7)}`,
        key: selectedKey,
        date: chosenDate
      });
      saveWechslerData(id, wData);

      const curIdx = ALL_WECHSLER_KEYS.indexOf(selectedKey);
      if (curIdx >= 0 && curIdx + 1 < ALL_WECHSLER_KEYS.length) {
        wechslerLessonSelect.value = ALL_WECHSLER_KEYS[curIdx + 1];
      }

      syncWechslerUi(id);
      refreshCurrentEvaluation();
    }

    document.getElementById('btnAddWechslerLesson').onclick = addWechslerLessonEntry;
    wechslerLessonDate.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') {
        e.preventDefault();
        addWechslerLessonEntry();
      }
    });

    document.getElementById('btnBulkFillGrund').onclick = () => {
      if (!lastInspected?.student?.id) return;
      const id = lastInspected.student.id;
      const wData = getWechslerData(id);
      const fillDate = wechslerLessonDate.value || wData.abnDate || '';

      if (!fillDate) {
        wechslerLessonDate.classList.add('input-invalid');
        wechslerLessonDate.focus();
        setTimeout(() => wechslerLessonDate.classList.remove('input-invalid'), 1800);
        return;
      }

      const existingKeys = new Set(wData.lessons.map(l => l.key));
      for (let i = 1; i <= 12; i++) {
        const k = `G${i}`;
        if (!existingKeys.has(k)) {
          wData.lessons.push({
            id: `wl_${Date.now()}_${i}_${Math.random().toString(36).slice(2, 6)}`,
            key: k,
            date: fillDate
          });
        }
      }
      wData.enabled = true;
      saveWechslerData(id, wData);
      syncWechslerUi(id);
      refreshCurrentEvaluation();
    };

    chkWechsler.onchange = () => {
      if (!lastInspected?.student?.id) return;
      const id = lastInspected.student.id;
      const wData = getWechslerData(id);
      wData.enabled = chkWechsler.checked;
      saveWechslerData(id, wData);
      syncWechslerUi(id);
      refreshCurrentEvaluation();
    };
    wechslerSchoolName.oninput = (e) => {
      e.stopPropagation();
      if (!lastInspected?.student?.id) return;
      const id = lastInspected.student.id;
      const wData = getWechslerData(id);
      wData.schoolName = wechslerSchoolName.value;
      saveWechslerData(id, wData);
      refreshCurrentEvaluation();
    };
    wechslerAbnDate.onchange = () => {
      if (!lastInspected?.student?.id) return;
      const id = lastInspected.student.id;
      const wData = getWechslerData(id);
      wData.abnDate = wechslerAbnDate.value;
      if (!wechslerLessonDate.value && wechslerAbnDate.value) {
        wechslerLessonDate.value = wechslerAbnDate.value;
      }
      saveWechslerData(id, wData);
      syncWechslerUi(id);
      refreshCurrentEvaluation();
    };
    document.getElementById('btnClearWechsler').onclick = () => {
      if (!lastInspected?.student?.id) return;
      const id = lastInspected.student.id;
      const wData = getWechslerData(id);
      wData.lessons = [];
      saveWechslerData(id, wData);
      syncWechslerUi(id);
      refreshCurrentEvaluation();
    };

    // 7. Auth & API Helpers
    fetch(`${baseOrigin}/bff/auth/me`, { headers: API_HEADERS, credentials: 'include' })
      .then(r => r.json())
      .then(me => {
        if (me.FirstName) document.getElementById('userFirst').textContent = me.FirstName;
        if (me.LastName) document.getElementById('userLast').textContent = me.LastName;
      })
      .catch(() => {});

    function extractRows(raw) {
      if (!raw || typeof raw !== 'object') return [];
      if (Array.isArray(raw.rows)) return raw.rows.map(r => (r && typeof r === 'object') ? (r.d || r.data || r) : r);
      if (Array.isArray(raw.items)) return raw.items;
      if (Array.isArray(raw.data)) return raw.data;
      if (Array.isArray(raw)) return raw;
      return [];
    }

    // ==========================================================================
    // 8. THEORIE-PARSER & REGELWERK (§ 4 FahrschAusbO)
    // ==========================================================================
    function normalizeTheoryItem(rawItem) {
      const item = (rawItem && typeof rawItem === 'object') ? (rawItem.data || rawItem.d || rawItem) : rawItem;
      if (!item || typeof item !== 'object') return { isoDate: '', datum: '-', zeit: '-', min: 0, units: 0, klasse: '', kapitel: '-', kapNum: null, lehrer: '-', filiale: '-', geloescht: false, isExternal: false };
      const isoMatch = String(item.datum || '').match(/^(\d{4})-(\d{2})-(\d{2})/);
      const isoDate = isoMatch ? `${isoMatch[1]}-${isoMatch[2]}-${isoMatch[3]}` : '';
      const datum = isoMatch ? `${isoMatch[3]}.${isoMatch[2]}.${isoMatch[1]}` : (item.datum || '-');
      const zeit = (item.von && item.bis) ? `${item.von} - ${item.bis}` : (item.von || '-');
      const min = Number(item.minuten) || 0;
      const units = min / 90;
      const klasse = String(item.klasse ?? '').trim();
      const kapitel = String(item.kapitel ?? '-').trim() || '-';
      const numMatch = kapitel.match(/^\s*(\d+)/);
      const kapNum = numMatch ? Number(numMatch[1]) : null;
      const lehrer = [item.fahrlehrerVorname, item.fahrlehrerNachname].filter(Boolean).join(' ').trim() || '-';
      const filiale = String(item.filiale ?? '-').trim() || '-';
      return { isoDate, datum, zeit, min, units, klasse, kapitel, kapNum, lehrer, filiale, geloescht: Boolean(item.geloescht), isExternal: false };
    }

    function evaluateTheoryRules(theory, studentId) {
      if (!theory || !theory.ok) return null;

      const extItems = buildExternalTheoryItems(studentId);
      const wData = getWechslerData(studentId);
      const allItems = [...theory.items, ...extItems];

      const ruleMode = document.getElementById('qRuleMode').value;
      const reqGrund = ruleMode === 'erweiterung' ? 6 : 12;
      const reqZusatz = 2;
      const maxDoublesAllowed = ruleMode === 'erweiterung' ? 0 : 2;

      const grundCounts = new Map();
      const grundExtFlags = new Map();
      const zusatzCounts = new Map();
      const zusatzExtFlags = new Map();
      const minutesByDate = new Map();
      let unnumberedGrundUnits = 0;
      let oldInternalLessonCount = 0;
      const expiredExternalItems = [];
      let missingDateExternalCount = 0;

      const twoYearsAgo = new Date();
      twoYearsAgo.setHours(0, 0, 0, 0);
      twoYearsAgo.setFullYear(twoYearsAgo.getFullYear() - 2);
      const cutoffIso = `${twoYearsAgo.getFullYear()}-${String(twoYearsAgo.getMonth() + 1).padStart(2, '0')}-${String(twoYearsAgo.getDate()).padStart(2, '0')}`;

      for (const it of allItems) {
        if (it.isoDate) {
          minutesByDate.set(it.isoDate, (minutesByDate.get(it.isoDate) || 0) + it.min);
        }

        const isExpired = Boolean(it.isoDate && it.isoDate < cutoffIso);
        if (isExpired) {
          if (it.isExternal) {
            expiredExternalItems.push(it);
            continue;
          } else {
            oldInternalLessonCount++;
          }
        }

        if (it.isExternal && !it.isoDate) {
          missingDateExternalCount++;
        }

        const isZusatz = Boolean(it.klasse) || (it.kapNum !== null && it.kapNum >= 13);
        if (isZusatz) {
          const zKey = it.kapNum !== null ? `${it.klasse || 'Zusatz'}-${it.kapNum}` : `${it.klasse || 'Zusatz'}-${it.kapitel}`;
          zusatzCounts.set(zKey, (zusatzCounts.get(zKey) || 0) + it.units);
          if (it.isExternal) zusatzExtFlags.set(zKey, true);
        } else if (it.kapNum !== null && it.kapNum >= 1 && it.kapNum <= 12) {
          grundCounts.set(it.kapNum, (grundCounts.get(it.kapNum) || 0) + it.units);
          if (it.isExternal) grundExtFlags.set(it.kapNum, true);
        } else {
          unnumberedGrundUnits += it.units;
        }
      }

      let uniqueGrundChapters = 0;
      const doubledChapters = [];
      const overRepeatedChapters = [];
      const missingGrundChapters = [];

      for (let ch = 1; ch <= 12; ch++) {
        const u = grundCounts.get(ch) || 0;
        if (u >= 1) {
          uniqueGrundChapters++;
          if (u >= 2) doubledChapters.push(ch);
          if (u > 2) overRepeatedChapters.push(ch);
        } else {
          missingGrundChapters.push(ch);
        }
      }

      const usableDoubles = Math.min(doubledChapters.length, maxDoublesAllowed);
      const effectiveGrundUnits = Math.min(reqGrund, uniqueGrundChapters + usableDoubles + unnumberedGrundUnits);
      const uniqueZusatzThemes = Array.from(zusatzCounts.entries()).filter(([, u]) => u >= 1).length;
      const effectiveZusatzUnits = Math.min(reqZusatz, uniqueZusatzThemes);

      const dailyViolations = [];
      for (const [iso, mins] of minutesByDate.entries()) {
        if (mins > 180) {
          const [y, m, d] = iso.split('-');
          dailyViolations.push(`${d}.${m}.${y} (${mins} Min.)`);
        }
      }

      const validExtCount = extItems.length - expiredExternalItems.length;
      const abnValidity = wData.abnDate ? checkTwoYearValidity(wData.abnDate) : null;
      const isAbnExpired = Boolean(wData.enabled && abnValidity && abnValidity.isExpired);

      const reasons = [];
      if (wData.enabled && (extItems.length > 0 || wData.abnDate)) {
        const abnText = wData.abnDate ? ` · ABN ausgestellt am ${formatGermanDate(wData.abnDate)}` : ' · ABN-Datum noch nicht hinterlegt';
        reasons.push(`Fahrschulwechsler (${wData.schoolName.trim() || 'Name nicht hinterlegt'}${abnText}): ${validExtCount} gültige DS${expiredExternalItems.length > 0 ? `, ${expiredExternalItems.length} abgelaufene DS` : ''}.`);
      }
      if (isAbnExpired) {
        reasons.push(`❌ Ausbildungsbescheinigung (ABN) der Vor-Fahrschule vom ${formatGermanDate(wData.abnDate)} ist abgelaufen (${abnValidity.label}).`);
      }
      if (expiredExternalItems.length > 0) {
        const expDesc = expiredExternalItems.map(x => `${x.kapitel.split('(')[0].trim()} (${x.datum})`).join(', ');
        reasons.push(`❌ Abgelaufene Fremdstunden (> 2 Jahre gem. § 4 Abs. 6 FahrschAusbO — nicht anrechenbar): ${expDesc}.`);
      }
      if (missingDateExternalCount > 0) {
        reasons.push(`⚠️ Bei ${missingDateExternalCount} eingetragener Fremdstunde(n) fehlt das Unterrichtsdatum zur Fristprüfung.`);
      }
      if (effectiveGrundUnits < reqGrund) {
        reasons.push(`Grundstoff unvollständig: ${effectiveGrundUnits} von ${reqGrund} DS angerechnet (Fehlende Lektionen: ${missingGrundChapters.join(', ')}; genutzte Dopplungen: ${usableDoubles}/${maxDoublesAllowed}).`);
      }
      if (doubledChapters.length > maxDoublesAllowed && ruleMode === 'ersterwerb') {
        reasons.push(`Zu viele Dopplungen im Grundstoff (Lektion ${doubledChapters.join(', ')}) — max. 2 Lektionen aus 1–12 dürfen doppelt angerechnet werden.`);
      }
      if (overRepeatedChapters.length > 0) {
        reasons.push(`Lektion ${overRepeatedChapters.join(', ')} wurde 3× oder häufiger besucht (max. 2× anrechenbar).`);
      }
      if (effectiveZusatzUnits < reqZusatz) {
        reasons.push(`Klassenspezifischer Zusatzstoff unvollständig: ${effectiveZusatzUnits} von ${reqZusatz} Themen (B-1 und B-2 sind beide Pflicht).`);
      }
      if (dailyViolations.length > 0) {
        reasons.push(`Tageslimit überschritten (§ 4 Abs. 1 FahrschAusbO erlaubt max. 180 Min./Tag): ${dailyViolations.join(', ')}.`);
      }
      if (oldInternalLessonCount > 0) {
        reasons.push(`${oldInternalLessonCount} eigene Unterrichtseinheit(en) überschreiten die 2-Jahres-Frist (§ 4 Abs. 6 FahrschAusbO).`);
      }

      const allowed = effectiveGrundUnits >= reqGrund
        && effectiveZusatzUnits >= reqZusatz
        && dailyViolations.length === 0
        && oldInternalLessonCount === 0
        && expiredExternalItems.length === 0
        && !isAbnExpired;

      const totalCombinedMin = theory.totalMin + validExtCount * 90;

      if (allowed) {
        reasons.push(`Grundstoff erfüllt: ${effectiveGrundUnits}/${reqGrund} DS (${uniqueGrundChapters} Themen${usableDoubles ? `, inkl. ${usableDoubles} zulässiger Dopplung: Lektion ${doubledChapters.slice(0, usableDoubles).join(', ')}` : ''}).`);
        reasons.push(`Zusatzstoff erfüllt: ${effectiveZusatzUnits}/${reqZusatz} klassenspezifische Themen vollständig.`);
        reasons.push(`Anrechenbare Gesamtzeit: ${totalCombinedMin} Min. (${theory.items.length + validExtCount} gültige Termine) — konform gemäß § 4 FahrschAusbO.`);
      }

      return {
        allowed,
        ruleModeLabel: ruleMode === 'erweiterung' ? 'Erweiterung (6 Grund + 2 Zusatz)' : 'Ersterwerb (12 Grund + 2 Zusatz)',
        reasons,
        effectiveGrundUnits,
        reqGrund,
        effectiveZusatzUnits,
        reqZusatz,
        grundCounts,
        grundExtFlags,
        zusatzCounts,
        zusatzExtFlags,
        allItems,
        extCount: extItems.length,
        validExtCount,
        expiredExtCount: expiredExternalItems.length,
        internalCount: theory.items.length,
        totalCombinedMin,
        wechslerSchool: wData.schoolName.trim() || 'Fremdfahrschule',
        wechslerAbnDate: wData.abnDate ? formatGermanDate(wData.abnDate) : ''
      };
    }

    async function fetchTheoryOnly(id, signal, force = false) {
      if (!force && theoryCache.has(id)) return theoryCache.get(id);
      const url = `${baseOrigin}/bff/api/v2/theoriestunden/kunde/${encodeURIComponent(id)}?skipDeleted=true&pagination.page=1&pagination.pageSize=100&pagination.count=100`;
      const res = await fetch(url, { headers: API_HEADERS, credentials: 'include', signal });
      if (!res.ok) {
        const errObj = { ok: false, error: `HTTP ${res.status}`, items: [], raw: { error: res.status } };
        theoryCache.set(id, errObj);
        return errObj;
      }
      const raw = await res.json();
      const items = extractRows(raw).map(normalizeTheoryItem).filter(it => !it.geloescht);
      let totalMin = 0, grundCount = 0, klasseCount = 0;
      for (const it of items) {
        totalMin += it.min;
        if (it.klasse || (it.kapNum !== null && it.kapNum >= 13)) klasseCount++;
        else grundCount++;
      }
      const result = { ok: true, raw, items, totalMin, grundCount, klasseCount };
      theoryCache.set(id, result);
      return result;
    }

    // ==========================================================================
    // 9. PRAXIS & PFLICHTFAHRTEN-ANALYSE ENGINE
    // ==========================================================================
    function normalizePraxisItem(rawItem) {
      const item = (rawItem && typeof rawItem === 'object') ? (rawItem.data || rawItem.d || rawItem) : rawItem;
      if (!item || typeof item !== 'object') {
        return { isoDate: '', sortTs: 0, datum: '-', startTime: '', zeit: '-', min: 0, kfz: '-', lehrer: '-', klasse: 'B', art: '-', bemerkung: '', betrag: 0, inhalt: '', hasGf: false, geloescht: false };
      }

      const rawDate = String(pickProp(item, 'datum', 'terminDatum', 'date', 'start') || '');
      const isoMatch = rawDate.match(/^(\d{4})-(\d{2})-(\d{2})/);
      const dmyMatch = rawDate.match(/^(\d{2})\.(\d{2})\.(\d{4})/);

      let isoDate = '';
      let datum = rawDate || '-';
      if (isoMatch) {
        isoDate = `${isoMatch[1]}-${isoMatch[2]}-${isoMatch[3]}`;
        datum = `${isoMatch[3]}.${isoMatch[2]}.${isoMatch[1]}`;
      } else if (dmyMatch) {
        isoDate = `${dmyMatch[3]}-${dmyMatch[2]}-${dmyMatch[1]}`;
        datum = `${dmyMatch[1]}.${dmyMatch[2]}.${dmyMatch[3]}`;
      }

      const von = String(pickProp(item, 'von', 'startZeit', 'beginn') || '').trim();
      const bis = String(pickProp(item, 'bis', 'endZeit', 'ende') || '').trim();
      const zeitDirect = String(pickProp(item, 'zeit', 'uhrzeit', 'zeitraum') || '').trim();
      const zeit = (von && bis) ? `${von} - ${bis}` : (zeitDirect || von || '-');

      const startTimeMatch = zeit.match(/(\d{1,2}):(\d{2})/);
      const startTime = startTimeMatch
        ? `${startTimeMatch[1].padStart(2, '0')}:${startTimeMatch[2]}`
        : '00:00';

      const sortTs = isoDate ? Date.parse(`${isoDate}T${startTime}:00`) || 0 : 0;

      const min = Number(pickProp(item, 'minuten', 'dauer', 'dauerInMinuten', 'minutes')) || 0;
      const kfz = String(pickProp(item, 'kfz', 'fahrzeug', 'kennzeichen', 'kfzKennung', 'fahrzeugKennung', 'kfzBezeichnung') ?? '-').trim() || '-';

      const flVor = pickProp(item, 'fahrlehrerVorname') || '';
      const flNach = pickProp(item, 'fahrlehrerNachname') || '';
      const lehrer = [flVor, flNach].filter(Boolean).join(' ').trim() || String(pickProp(item, 'fahrlehrer', 'fahrlehrerName', 'lehrer') ?? '-').trim();

      const klasse = String(pickProp(item, 'klasse', 'klassen', 'fuehrerscheinklasse') ?? 'B').trim();
      const art = String(pickProp(item, 'art', 'leistungsart', 'fahrstundenArt', 'bezeichnung', 'typ', 'leistungsBezeichnung') ?? 'Übungsfahrt').trim();
      const bemerkung = String(pickProp(item, 'bemerkung', 'kommentar', 'notiz', 'hinweis') ?? '').trim();
      const betrag = Number(pickProp(item, 'betrag', 'preis', 'summe', 'brutto') ?? 0) || 0;

      let inhalt = String(pickProp(item, 'inhalt', 'ausbildungsinhalt', 'thema', 'kurzbezeichnung', 'ausbildungsInhalte') ?? '').trim();
      if (!inhalt) {
        for (const [k, v] of Object.entries(item)) {
          if (typeof v === 'string' && /\/?gf\.?/i.test(v) && k.toLowerCase() !== 'art') {
            inhalt = v.trim();
            break;
          }
        }
      }

      const hasGf = /\/?\bgf\b\.?/i.test(inhalt) || /\/?\bgf\b\.?/i.test(bemerkung);

      return {
        isoDate,
        sortTs,
        datum,
        startTime,
        zeit,
        min,
        kfz,
        lehrer,
        klasse,
        art,
        bemerkung,
        betrag,
        inhalt: inhalt || (hasGf ? '/Gf.' : ''),
        hasGf,
        geloescht: Boolean(item.geloescht)
      };
    }

    async function fetchPraxisOnly(id, signal, force = false) {
      if (!force && praxisCache.has(id)) return praxisCache.get(id);

      const candidateUrls = [
        `${baseOrigin}/bff/api/v2/fahrstunden/kunde/${encodeURIComponent(id)}?skipDeleted=true&pagination.page=1&pagination.pageSize=100&pagination.count=100`,
        `${baseOrigin}/bff/api/v1/fahrstunden/kunde/${encodeURIComponent(id)}?skipDeleted=true&pagination.page=1&pagination.pageSize=100&pagination.count=100`,
        `${baseOrigin}/bff/api/v3/fahrstunden/kunde/${encodeURIComponent(id)}?skipDeleted=true&pagination.page=1&pagination.pageSize=100&pagination.count=100`
      ];

      let raw = null;
      let lastStatus = '404';

      for (const url of candidateUrls) {
        try {
          const res = await fetch(url, { headers: API_HEADERS, credentials: 'include', signal });
          if (res.ok) {
            raw = await res.json();
            break;
          }
          lastStatus = String(res.status);
        } catch (e) {
          if (e.name === 'AbortError') throw e;
          lastStatus = e.message;
        }
      }

      if (!raw) {
        const errObj = { ok: false, error: `HTTP ${lastStatus}`, items: [], raw: { error: lastStatus } };
        praxisCache.set(id, errObj);
        return errObj;
      }

      const items = extractRows(raw).map(normalizePraxisItem).filter(it => !it.geloescht);
      const totalMin = items.reduce((acc, it) => acc + it.min, 0);
      const totalBetrag = items.reduce((acc, it) => acc + it.betrag, 0);

      const result = { ok: true, raw, items, totalMin, totalBetrag };
      praxisCache.set(id, result);
      return result;
    }

    function autoDetectPraxisConfig(student, kartei, praxis) {
      const erteilungRaw = String(pickProp(kartei, 'fiderteilungsart') || '').toUpperCase();
      const isUmschreiber = erteilungRaw === 'U';

      const kB197 = Boolean(pickProp(kartei, 'b197')) || Boolean(pickProp(kartei, 'b197Testfahrt'));
      const classHas197 = (student?._klassenStr || '').includes('197');
      const hasTestfahrtLesson = (praxis?.items || []).some(it => /197/i.test(it.art));
      const isB197 = kB197 || classHas197 || hasTestfahrtLesson;

      return {
        erwerbMode: isUmschreiber ? 'umschreiber' : 'neuerwerber',
        b197Mode: isB197 ? 'yes' : 'no',
        autoNote: `Automatisch erkannt aus Kartei: ${isUmschreiber ? 'Umschreiber (U)' : 'Neuerwerber / Ersterwerb (E)'} · ${isB197 ? 'Mit B197' : 'Ohne B197'}`
      };
    }

    function evaluatePraxisRules(praxis, erwerbMode, b197Mode, potsdamMode = isPotsdamMode()) {
      if (!praxis || !praxis.ok) return null;

      const isNeuerwerber = erwerbMode === 'neuerwerber';
      const isB197 = b197Mode === 'yes';

      const chronItems = [...praxis.items].sort((a, b) => a.sortTs - b.sortTs);

      const uebungsfahrten = [];
      const ueberlandList = [];
      const autobahnList = [];
      const nachtList = [];
      const testfahrtB197List = [];
      const pruefungsList = [];
      const sonderfahrtenAll = [];

      for (const it of chronItems) {
        const aLower = it.art.toLowerCase();
        if (aLower.includes('überland') || aLower.includes('ueberland')) {
          ueberlandList.push(it);
          sonderfahrtenAll.push(it);
        } else if (aLower.includes('autobahn')) {
          autobahnList.push(it);
          sonderfahrtenAll.push(it);
        } else if (aLower.includes('nacht') || aLower.includes('beleuchtung') || aLower.includes('dämmerung')) {
          nachtList.push(it);
          sonderfahrtenAll.push(it);
        } else if (aLower.includes('197') || aLower.includes('testfahrt')) {
          testfahrtB197List.push(it);
        } else if (aLower.includes('prüfung') || aLower.includes('pruefung')) {
          pruefungsList.push(it);
        } else {
          uebungsfahrten.push(it);
        }
      }

      // 1. SONDERFAHRTEN-CHECK (inkl. Potsdam-Schalter für AB An-/Abfahrt)
      const uelMin = ueberlandList.reduce((s, x) => s + x.min, 0);
      const uelHas225Block = ueberlandList.some(x => x.min === 225);
      const uelOk = !isNeuerwerber || uelMin >= 225;

      const abMin = autobahnList.reduce((s, x) => s + x.min, 0);
      const abHas180Block = autobahnList.some(x => x.min === 180);
      const abDates = new Set(autobahnList.map(x => x.isoDate).filter(Boolean));

      const twentyMinDrives = uebungsfahrten.filter(x => x.min === 20);
      const twentyMinOnAbDate = twentyMinDrives.filter(x => abDates.has(x.isoDate));
      const abAnAbfahrtOk = !isNeuerwerber || potsdamMode || twentyMinOnAbDate.length >= 2 || twentyMinDrives.length >= 2;
      const abOk = !isNeuerwerber || (abMin >= 180 && abAnAbfahrtOk);

      const nfMin = nachtList.reduce((s, x) => s + x.min, 0);
      const nfHas135Block = nachtList.some(x => x.min === 135);
      const nfOk = !isNeuerwerber || nfMin >= 135;

      const sonderAllOk = uelOk && abOk && nfOk;

      // 2. B197-CHECK
      const testfahrt15 = testfahrtB197List.filter(x => x.min >= 15);
      const schaltKfzSet = new Set(testfahrtB197List.map(x => x.kfz).filter(k => k && k !== '-' && k !== 'AUD'));
      const ninetyMinDrives = uebungsfahrten.filter(x => x.min === 90);
      ninetyMinDrives.forEach(x => { if (x.kfz && x.kfz !== '-' && x.kfz !== 'AUD') schaltKfzSet.add(x.kfz); });

      const sixtyMinDrives = uebungsfahrten.filter(x => x.min === 60);
      const sixtyMinOnSchaltKfz = schaltKfzSet.size > 0
        ? sixtyMinDrives.filter(x => schaltKfzSet.has(x.kfz))
        : sixtyMinDrives;

      const b197NinetyOk = !isB197 || ninetyMinDrives.length >= 5;
      const b197SixtyOk = !isB197 || !isNeuerwerber || sixtyMinOnSchaltKfz.length >= 1 || sixtyMinDrives.length >= 1;
      const b197TestOk = !isB197 || testfahrt15.length >= 1;
      const b197AllOk = b197NinetyOk && b197SixtyOk && b197TestOk;

      // 3. GRUNDFAHRAUFGABEN (/Gf.) & CHRONOLOGIE-VERBOT NACH SONDERFAHRT
      const gfLessons = chronItems.filter(x => x.hasGf);
      const firstSonderfahrt = sonderfahrtenAll.length > 0 ? sonderfahrtenAll[0] : null;

      const gfAfterSonderViolations = [];
      if (firstSonderfahrt && firstSonderfahrt.sortTs > 0) {
        for (const gf of gfLessons) {
          if (gf.sortTs > firstSonderfahrt.sortTs) {
            gfAfterSonderViolations.push({
              gfLesson: gf,
              sonderLesson: firstSonderfahrt
            });
          }
        }
      }

      const gfCountOk = !isNeuerwerber || gfLessons.length >= 5;
      const gfChronologyOk = !isNeuerwerber || gfAfterSonderViolations.length === 0;
      const gfAllOk = gfCountOk && gfChronologyOk;

      // 4. UNGEWÖHNLICHE MINUTENWERTE
      const standardMins = new Set([20, 45, 60, 80, 90, 120, 135, 180]);
      const oddMinuteDrives = uebungsfahrten.filter(x => !standardMins.has(x.min));

      const allowed = sonderAllOk && b197AllOk && gfAllOk;
      const reasons = [];

      if (isNeuerwerber) {
        if (!uelOk) reasons.push(`Überlandfahrt fehlt/unvollständig: ${uelMin} / 225 Min. eingetragen (Soll: 1× 225 Min.).`);
        if (abMin < 180) reasons.push(`Autobahnfahrt fehlt/unvollständig: ${abMin} / 180 Min. eingetragen (Soll: 1× 180 Min.).`);
        if (!abAnAbfahrtOk) reasons.push(`An- und Abfahrt zur Autobahnfahrt fehlt: Nur ${twentyMinOnAbDate.length}× 20 Min. Übungsfahrt am Tag der Autobahnfahrt gefunden (Soll: 2× 20 Min. An-/Abfahrt).`);
        if (!nfOk) reasons.push(`Nachtfahrt fehlt/unvollständig: ${nfMin} / 135 Min. eingetragen (Soll: 1× 135 Min.).`);

        if (!gfCountOk) {
          reasons.push(`Zu wenige Grundfahraufgaben (/Gf.): Nur ${gfLessons.length}× /Gf. markiert (Pflicht bei Neuerwerbern: mind. 5× /Gf.).`);
        }
        if (!gfChronologyOk) {
          gfAfterSonderViolations.forEach(v => {
            reasons.push(`❌ CHRONOLOGIE-VERSTOSS (/Gf. nach Sonderfahrt): Die Fahrstunde mit /Gf. am ${v.gfLesson.datum} (${v.gfLesson.zeit}) liegt zeitlich NACH der Sonderfahrt (${v.sonderLesson.art} am ${v.sonderLesson.datum})!`);
          });
        }
      } else {
        reasons.push(`Umschreiber-Modus aktiv: Keine Sonderfahrten (ÜL/AB/NF) und keine /Gf.-Mindestanzahl erforderlich.`);
      }

      if (isB197) {
        if (!b197NinetyOk) reasons.push(`B197 Schaltstunden unvollständig: ${ninetyMinDrives.length} / 5 Fahrten à 90 Min. eingetragen.`);
        if (!b197SixtyOk) reasons.push(`B197 Begleitstunde (1× 60 Min. vor Testfahrt) fehlt.`);
        if (!b197TestOk) reasons.push(`B197 Testfahrt fehlt: Keine „Testfahrt B197“ (15 Min.) eingetragen.`);
      }

      if (allowed) {
        if (isNeuerwerber) {
          const abSubNote = potsdamMode
            ? 'Filiale Potsdam: ohne 2× 20 Min. An-/Abfahrt'
            : `inkl. ${twentyMinOnAbDate.length}× 20 Min. An-/Abfahrt`;
          reasons.push(`Sonderfahrten komplett: 1× 225 Min. Überland, 1× 180 Min. Autobahn (${abSubNote}) & 1× 135 Min. Nachtfahrt.`);
          reasons.push(`Grundfahraufgaben (/Gf.) korrekt: ${gfLessons.length}× /Gf. eingetragen — alle zeitlich vor der ersten Sonderfahrt (${firstSonderfahrt ? firstSonderfahrt.datum : '—'}).`);
        }
        if (isB197) {
          const kfzInfo = schaltKfzSet.size ? ` auf KFZ ${Array.from(schaltKfzSet).join(', ')}` : '';
          reasons.push(`B197 vollständig: ${ninetyMinDrives.length}× 90 Min.${kfzInfo}, 1× 60 Min. + 1× 15 Min. Testfahrt B197 korrekt dokumentiert.`);
        }
      }

      return {
        allowed,
        isNeuerwerber,
        isB197,
        potsdamMode,
        reasons,
        uelMin, uelHas225Block, uelOk,
        abMin, abHas180Block, twentyMinOnAbDate, twentyMinDrives, abAnAbfahrtOk, abOk,
        nfMin, nfHas135Block, nfOk,
        sonderAllOk,
        ninetyMinDrives, sixtyMinDrives, sixtyMinOnSchaltKfz, testfahrt15, schaltKfzSet,
        b197NinetyOk, b197SixtyOk, b197TestOk, b197AllOk,
        gfLessons, firstSonderfahrt, gfAfterSonderViolations, gfCountOk, gfChronologyOk, gfAllOk,
        oddMinuteDrives,
        pruefungsList
      };
    }

    function renderPraxisModuleTab(student, praxis) {
      const configBar = document.getElementById('praxisConfigBar');
      const verdictBox = document.getElementById('praxisVerdictBox');
      const cardsGrid = document.getElementById('praxisCheckCards');
      const pBody = document.getElementById('praxisBody');
      const pFoot = document.getElementById('praxisFoot');
      const btnPrintPraxis = document.getElementById('btnPrintPraxis');

      document.getElementById('praxisTabTitle').textContent =
        `2. Praxis & Pflichtfahrten (${student._nachname}, ${student._vorname} · ${student._karteiNr || '—'})`;

      if (!praxis || !praxis.ok) {
        configBar.style.display = 'none';
        cardsGrid.style.display = 'none';
        pFoot.style.display = 'none';
        btnPrintPraxis.style.display = 'none';
        verdictBox.innerHTML = `<div class="verdict-box no"><div class="verdict-headline">Fehler beim Laden der Fahrstunden (${praxis?.error || 'Unbekannt'})</div></div>`;
        pBody.innerHTML = `<tr><td colspan="11" style="color:var(--err-solid);">Keine Fahrstunden-Antwort vom Server erhalten.</td></tr>`;
        return;
      }

      configBar.style.display = 'block';
      const erwerbMode = document.getElementById('praxisErwerbMode').value;
      const b197Mode = document.getElementById('praxisB197Mode').value;
      const ev = evaluatePraxisRules(praxis, erwerbMode, b197Mode, isPotsdamMode());
      if (lastInspected) lastInspected.evalPraxis = ev;

      const workerName = workerInput.value.trim() || '—';
      btnPrintPraxis.style.display = 'inline-flex';

      verdictBox.innerHTML = `
        <div class="verdict-box ${ev.allowed ? 'yes' : 'no'}">
          <div style="flex:1; min-width:280px;">
            <div class="verdict-headline">
              ${ev.allowed ? '✓ JA — PFLICHTFAHRTEN & CHRONOLOGIE KORREKT' : '✕ NICHT ZUGELASSEN — FEHLER BEI PFLICHTFAHRTEN'}
            </div>
            <ul class="verdict-reasons">
              ${ev.reasons.map(r => `<li>${r}</li>`).join('')}
            </ul>
          </div>
          <div class="verdict-stats">
            <div class="stat-col">
              <span class="stat-lbl">Kategorie</span>
              <span class="stat-val">${ev.isNeuerwerber ? 'Neuerwerber' : 'Umschreiber'}${ev.isB197 ? ' · B197' : ''}${ev.potsdamMode ? ' · Potsdam' : ''}</span>
            </div>
            <div class="stat-col">
              <span class="stat-lbl">Grundfahraufg. (/Gf.)</span>
              <span class="stat-val">${ev.gfLessons.length}${ev.isNeuerwerber ? ' / 5+' : ''}</span>
            </div>
            <div class="stat-col">
              <span class="stat-lbl">Gesamtminuten</span>
              <span class="stat-val">${praxis.totalMin} Min.</span>
            </div>
            <div class="stat-col">
              <span class="stat-lbl">Prüfer</span>
              <span class="stat-val" style="font-size:12.5px;">${escapeHtml(workerName)}</span>
            </div>
          </div>
        </div>
      `;

      const sixtyMinList = ev.sixtyMinDrives || [];
      const abAnAbfBadgeHtml = ev.potsdamMode
        ? `<span class="fsm-badge b-ext">✓ Frei (Filiale Potsdam)</span>`
        : `<span class="fsm-badge ${ev.abAnAbfahrtOk ? 'b-yes' : 'b-no'}">${ev.twentyMinOnAbDate.length}× am AB-Tag (${ev.twentyMinDrives.length}× ges.)</span>`;

      cardsGrid.style.display = 'grid';
      cardsGrid.innerHTML = `
        <div class="pcheck-card">
          <div class="pcheck-hdr">
            <span>1. Sonderfahrten (Anlage 4)</span>
            <span class="fsm-badge ${ev.sonderAllOk ? 'b-yes' : 'b-no'}">${ev.isNeuerwerber ? (ev.sonderAllOk ? '✓ Erfüllt' : '✕ Fehlt') : 'Umschreiber (Frei)'}</span>
          </div>
          <div class="pcheck-item">
            <span>Überlandfahrt (1× 225 Min.)</span>
            <span class="fsm-badge ${ev.uelOk ? 'b-yes' : 'b-no'}">${ev.uelMin} / 225 Min.</span>
          </div>
          <div class="pcheck-item">
            <span>Autobahnfahrt (1× 180 Min.)</span>
            <span class="fsm-badge ${ev.abMin >= 180 || !ev.isNeuerwerber ? 'b-yes' : 'b-no'}">${ev.abMin} / 180 Min.</span>
          </div>
          <div class="pcheck-item">
            <span>↳ An- & Abfahrt AB (2× 20 Min.)</span>
            ${abAnAbfBadgeHtml}
          </div>
          <div class="pcheck-item">
            <span>Nachtfahrt (1× 135 Min.)</span>
            <span class="fsm-badge ${ev.nfOk ? 'b-yes' : 'b-no'}">${ev.nfMin} / 135 Min.</span>
          </div>
        </div>

        <div class="pcheck-card">
          <div class="pcheck-hdr">
            <span>2. B197 Schaltkompetenz</span>
            <span class="fsm-badge ${ev.b197AllOk ? 'b-yes' : 'b-no'}">${ev.isB197 ? (ev.b197AllOk ? '✓ Erfüllt' : '✕ Unvollständig') : 'Nicht benötigt'}</span>
          </div>
          <div class="pcheck-item">
            <span>Schaltstunden (5× 90 Min.)</span>
            <span class="fsm-badge ${ev.b197NinetyOk ? 'b-yes' : 'b-no'}">${ev.ninetyMinDrives.length} / 5 (à 90m)</span>
          </div>
          <div class="pcheck-item">
            <span>Übungsstunde (1× 60 Min.)</span>
            <span class="fsm-badge ${ev.b197SixtyOk ? 'b-yes' : 'b-no'}">${ev.sixtyMinOnSchaltKfz.length >= 1 ? '✓ Vorhanden' : (sixtyMinList.length ? '✓ (60m vorh.)' : '✕ Fehlt')}</span>
          </div>
          <div class="pcheck-item">
            <span>Testfahrt B197 (1× 15 Min.)</span>
            <span class="fsm-badge ${ev.b197TestOk ? 'b-yes' : 'b-no'}">${ev.testfahrt15.length} / 1 (${ev.testfahrt15[0]?.min || 0}m)</span>
          </div>
          <div class="pcheck-item">
            <span>Erkanntes Schalt-KFZ</span>
            <span style="font-weight:600;">${ev.schaltKfzSet.size ? Array.from(ev.schaltKfzSet).join(', ') : '—'}</span>
          </div>
        </div>

        <div class="pcheck-card">
          <div class="pcheck-hdr">
            <span>3. Grundfahraufgaben (/Gf.)</span>
            <span class="fsm-badge ${ev.gfAllOk ? 'b-yes' : 'b-no'}">${ev.gfAllOk ? '✓ Korrekt' : '✕ Fehler'}</span>
          </div>
          <div class="pcheck-item">
            <span>Anzahl /Gf. Markierungen</span>
            <span class="fsm-badge ${ev.gfCountOk ? 'b-yes' : 'b-no'}">${ev.gfLessons.length} / mind. 5</span>
          </div>
          <div class="pcheck-item">
            <span>1. Sonderfahrt (Stichtag)</span>
            <span style="font-weight:600;">${ev.firstSonderfahrt ? `${ev.firstSonderfahrt.datum} (${ev.firstSonderfahrt.art})` : 'Noch keine'}</span>
          </div>
          <div class="pcheck-item">
            <span>/Gf. nach Sonderfahrt?</span>
            <span class="fsm-badge ${ev.gfChronologyOk ? 'b-yes' : 'b-no'}">
              ${ev.gfChronologyOk ? '✓ 0 Verstöße (Vor Sonderf.)' : `✕ ${ev.gfAfterSonderViolations.length} VERSTOSS!`}
            </span>
          </div>
          <div class="pcheck-item">
            <span>Prüfungsfahrt(en)</span>
            <span style="font-weight:600;">${ev.pruefungsList.length ? `${ev.pruefungsList.length}× (${ev.pruefungsList.map(p => p.bemerkung || p.datum).join(', ')})` : '—'}</span>
          </div>
        </div>
      `;

      const violationSet = new Set(ev.gfAfterSonderViolations.map(v => v.gfLesson));
      const abDates = new Set(
        praxis.items.filter(x => x.art.toLowerCase().includes('autobahn')).map(x => x.isoDate)
      );

      if (praxis.items.length === 0) {
        pBody.innerHTML = `<tr><td colspan="11" style="color:var(--text-muted);">Keine Fahrstunden für diesen Schüler eingetragen.</td></tr>`;
        pFoot.style.display = 'none';
      } else {
        pFoot.style.display = 'table-footer-group';
        document.getElementById('praxisSumMin').textContent = String(praxis.totalMin);
        document.getElementById('praxisSumBetrag').textContent =
          praxis.totalBetrag.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) + ' €';

        pBody.innerHTML = praxis.items.map(it => {
          const isViolation = violationSet.has(it);
          const aLower = it.art.toLowerCase();
          const isSonder = aLower.includes('überland') || aLower.includes('ueberland') || aLower.includes('autobahn') || aLower.includes('nacht');
          const isTest197 = aLower.includes('197');
          const isAnAbfahrt20 = !ev.potsdamMode && !isSonder && it.min === 20 && abDates.has(it.isoDate);
          const isB197Schalt90 = !isSonder && it.min === 90 && ev.schaltKfzSet.has(it.kfz);

          let hintBadge = '';
          if (isViolation) {
            hintBadge = `<span class="fsm-badge b-no">❌ VERBOTEN: /Gf. nach Sonderfahrt!</span>`;
          } else if (isSonder) {
            hintBadge = `<span class="fsm-badge b-yes">Sonderfahrt (${it.min}m)</span>`;
          } else if (isAnAbfahrt20) {
            hintBadge = `<span class="fsm-badge b-yes">✓ 20m An-/Abfahrt AB</span>`;
          } else if (isTest197) {
            hintBadge = `<span class="fsm-badge b-ext">✓ Testfahrt B197</span>`;
          } else if (isB197Schalt90) {
            hintBadge = `<span class="fsm-badge b-ext">B197 Schalter (90m)</span>`;
          } else if (it.hasGf) {
            hintBadge = `<span class="fsm-badge b-yes">✓ Grundfahraufgabe (/Gf.)</span>`;
          }

          const betragStr = it.betrag.toLocaleString('de-DE', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) + ' €';

          return `
            <tr class="${isViolation ? 'row-violation' : (isSonder ? 'row-sonder' : '')}">
              <td>${escapeHtml(it.datum)}</td>
              <td>${escapeHtml(it.zeit)}</td>
              <td style="text-align:right; font-weight:600;">${it.min}</td>
              <td><strong>${escapeHtml(it.kfz)}</strong></td>
              <td>${escapeHtml(it.lehrer)}</td>
              <td>${escapeHtml(it.klasse)}</td>
              <td><strong>${escapeHtml(it.art)}</strong></td>
              <td>${escapeHtml(it.bemerkung)}</td>
              <td style="text-align:right;">${betragStr}</td>
              <td><strong>${escapeHtml(it.inhalt)}</strong></td>
              <td>${hintBadge}</td>
            </tr>
          `;
        }).join('');
      }
    }

    document.getElementById('praxisErwerbMode').onchange = () => {
      if (lastInspected?.praxis) {
        renderPraxisModuleTab(lastInspected.student, lastInspected.praxis);
        updateRowBadges(lastInspected.student.id);
      }
    };
    document.getElementById('praxisB197Mode').onchange = () => {
      if (lastInspected?.praxis) {
        renderPraxisModuleTab(lastInspected.student, lastInspected.praxis);
        updateRowBadges(lastInspected.student.id);
      }
    };

    function renderKarteiModuleTab(student, k) {
      document.getElementById('karteiTabTitle').textContent = `3. Schüler-Kartei & Büro-Notizen: ${student._nachname}, ${student._vorname} (${student._karteiNr || '—'})`;
      document.getElementById('karteiTabLinks').innerHTML = `
        <a class="btn btn-outline btn-sm" style="background:#fff; color:#00394d;" href="${baseOrigin}/schueler/${student.id}/leistungen" target="_blank">Leistungen öffnen ↗</a>
        <a class="btn btn-outline btn-sm" style="background:#fff; color:#00394d;" href="${baseOrigin}/schueler/${student.id}" target="_blank">Stammdaten bearbeiten ↗</a>
      `;
      const tabBody = document.getElementById('karteiTabBody');
      tabBody.innerHTML = `<div class="kartei-grid">${buildKarteiCardsHtml(k, student)}</div>`;
      bindKarteiNoteEvents(tabBody, student);
    }

    function updateRowBadges(studentId) {
      const rowEl = document.getElementById(`row-${studentId}`);
      if (!rowEl) return;
      const th = theoryCache.get(studentId);
      const pr = praxisCache.get(studentId);
      const studentObj = allStudents.find(s => s.id === studentId);

      const nameCell = rowEl.querySelector('.student-name-cell');
      if (nameCell && studentObj) {
        nameCell.innerHTML = `${escapeHtml(studentObj._nachname)}, ${escapeHtml(studentObj._vorname)}${NotesManager.badgeHtml(studentId)}`;
      }

      if (th && th.ok) {
        const evT = evaluateTheoryRules(th, studentId);
        rowEl.querySelector('.verdict-theory-cell').innerHTML = evT && evT.allowed
          ? `<span class="fsm-badge b-yes">✓ THEORIE OK</span>`
          : `<span class="fsm-badge b-no">✕ THEORIE FEHLT</span>`;
      }
      if (pr && pr.ok) {
        let erwerbMode, b197Mode;
        if (lastInspected?.student?.id === studentId) {
          erwerbMode = document.getElementById('praxisErwerbMode').value;
          b197Mode = document.getElementById('praxisB197Mode').value;
        } else {
          const detected = autoDetectPraxisConfig(studentObj, StudentStore.karteiMap.get(studentId), pr);
          erwerbMode = detected.erwerbMode;
          b197Mode = detected.b197Mode;
        }
        const evP = evaluatePraxisRules(pr, erwerbMode, b197Mode, isPotsdamMode());
        rowEl.querySelector('.verdict-praxis-cell').innerHTML = evP && evP.allowed
          ? `<span class="fsm-badge b-yes">✓ PRAXIS OK</span>`
          : `<span class="fsm-badge b-no">✕ PRAXIS FEHLER</span>`;
      }
    }

    NotesManager.listeners.add(() => {
      if (lastInspected?.student?.id) {
        updateRowBadges(lastInspected.student.id);
      }
    });

    function refreshCurrentEvaluation() {
      if (!lastInspected) return;
      const { student, label, theory, praxis, kartei } = lastInspected;
      renderTheoryDetails(student, label, theory, praxis, kartei);
      if (student?.id) updateRowBadges(student.id);
    }

    function renderTheoryDetails(student, label, theory, praxis, kartei) {
      const studentId = student?.id || '';
      const evalTheory = evaluateTheoryRules(theory, studentId);

      if (!lastInspected || lastInspected.student?.id !== studentId) {
        const detected = autoDetectPraxisConfig(student, kartei, praxis);
        document.getElementById('praxisErwerbMode').value = detected.erwerbMode;
        document.getElementById('praxisB197Mode').value = detected.b197Mode;
        document.getElementById('praxisAutoDetectInfo').textContent = detected.autoNote;
        wechslerLessonDate.value = getWechslerData(studentId).abnDate || '';
      }

      const evalPraxis = evaluatePraxisRules(
        praxis,
        document.getElementById('praxisErwerbMode').value,
        document.getElementById('praxisB197Mode').value,
        isPotsdamMode()
      );

      lastInspected = { student, label, theory, praxis, evalTheory, evalPraxis, kartei };

      if (kartei) {
        try { renderKarteiModuleTab(student, kartei); } catch (e) { console.error('[Kartei Render Error]', e); }
      }
      if (praxis) {
        try { renderPraxisModuleTab(student, praxis); } catch (e) { console.error('[Praxis Render Error]', e); }
      }

      document.getElementById('detailTitle').textContent = `1. Theorie-Prüfungsreife (${label} · ${student?._karteiNr || '—'})`;
      document.getElementById('wechslerContainer').style.display = 'block';
      syncWechslerUi(studentId);

      document.getElementById('jsonView').textContent = JSON.stringify({
        theoriestunden: theory?.raw || theory,
        fahrstunden: praxis?.raw || praxis,
        kartei
      }, null, 2);

      const verdictBox = document.getElementById('verdictBox');
      const fileNoticeBox = document.getElementById('fileNoticeBox');
      const btnPrintTheory = document.getElementById('btnPrintTheory');
      const matrixEl = document.getElementById('chapterMatrix');
      const tBody = document.getElementById('theoryBody');

      if (!theory || !theory.ok) {
        verdictBox.innerHTML = `<div class="verdict-box no"><div class="verdict-headline">Abruffehler (${theory?.error})</div></div>`;
        fileNoticeBox.style.display = 'none';
        btnPrintTheory.style.display = 'none';
        matrixEl.innerHTML = '';
        tBody.innerHTML = '';
        return;
      }

      const workerName = workerInput.value.trim() || '—';
      verdictBox.innerHTML = `
        <div class="verdict-box ${evalTheory.allowed ? 'yes' : 'no'}">
          <div style="flex:1; min-width:280px;">
            <div class="verdict-headline">
              ${evalTheory.allowed ? '✓ JA — ZUR THEORIEPRÜFUNG ZUGELASSEN' : '✕ NICHT ZUGELASSEN'}
            </div>
            <ul class="verdict-reasons">
              ${evalTheory.reasons.map(r => `<li>${r}</li>`).join('')}
            </ul>
          </div>
          <div class="verdict-stats">
            <div class="stat-col">
              <span class="stat-lbl">Grundstoff</span>
              <span class="stat-val">${evalTheory.effectiveGrundUnits} / ${evalTheory.reqGrund} DS</span>
            </div>
            <div class="stat-col">
              <span class="stat-lbl">Zusatzstoff</span>
              <span class="stat-val">${evalTheory.effectiveZusatzUnits} / ${evalTheory.reqZusatz} DS</span>
            </div>
            <div class="stat-col">
              <span class="stat-lbl">Summe Minuten</span>
              <span class="stat-val">${evalTheory.totalCombinedMin} Min.</span>
            </div>
            <div class="stat-col">
              <span class="stat-lbl">Prüfer</span>
              <span class="stat-val" style="font-size:12.5px;">${escapeHtml(workerName)}</span>
            </div>
          </div>
        </div>
      `;

      fileNoticeBox.style.display = 'flex';
      btnPrintTheory.style.display = 'inline-flex';

      const pills = [];
      for (let ch = 1; ch <= 12; ch++) {
        const cnt = evalTheory.grundCounts.get(ch) || 0;
        const isExt = evalTheory.grundExtFlags.get(ch);
        const extSuffix = isExt ? ' [Wechsler]' : '';
        if (cnt >= 2) pills.push(`<span class="m-pill ${isExt ? 'm-ext' : 'm-double'}">Lektion ${ch} (${cnt}×)${extSuffix}</span>`);
        else if (cnt >= 1) pills.push(`<span class="m-pill ${isExt ? 'm-ext' : 'm-done'}">Lektion ${ch} ✓${extSuffix}</span>`);
        else pills.push(`<span class="m-pill m-miss">Lektion ${ch} fehlt</span>`);
      }
      const zEntries = Array.from(evalTheory.zusatzCounts.entries());
      if (zEntries.length === 0) pills.push(`<span class="m-pill m-miss">Zusatz B-1/B-2 fehlt</span>`);
      else {
        for (const [k, cnt] of zEntries) {
          const isExt = evalTheory.zusatzExtFlags.get(k);
          pills.push(`<span class="m-pill ${isExt ? 'm-ext' : 'm-done'}">${k} (${cnt}×) ✓${isExt ? ' [Wechsler]' : ''}</span>`);
        }
        if (zEntries.length < 2) pills.push(`<span class="m-pill m-miss">2. Zusatzthema fehlt</span>`);
      }
      matrixEl.innerHTML = pills.join('');

      tBody.innerHTML = evalTheory.allItems.length === 0
        ? `<tr><td colspan="7" style="color:var(--text-muted);">Keine Theoriestunden erfasst.</td></tr>`
        : evalTheory.allItems.map(t => {
            const isExtExpired = Boolean(t.isExternal && t.validity?.isExpired);
            const rowCls = isExtExpired ? 'row-violation' : (t.isExternal ? 'row-external' : '');
            const filialeHtml = t.isExternal
              ? `<span class="fsm-badge ${isExtExpired ? 'b-no' : 'b-ext'}">${isExtExpired ? '❌ ABGELAUFEN · ' : ''}${escapeHtml(t.filiale)}</span>`
              : escapeHtml(t.filiale);
            return `
              <tr class="${rowCls}">
                <td>${escapeHtml(t.datum)}</td>
                <td>${escapeHtml(t.zeit)}</td>
                <td style="text-align:right;">${t.min}</td>
                <td>${escapeHtml(t.klasse || '')}</td>
                <td>${escapeHtml(t.kapitel)}</td>
                <td>${t.isExternal ? `<span class="fsm-badge b-ext">Wechsler: ${escapeHtml(t.lehrer)}</span>` : escapeHtml(t.lehrer)}</td>
                <td>${filialeHtml}</td>
              </tr>
            `;
          }).join('');
    }

    // 10. DRUCKFUNKTIONEN
    document.getElementById('btnPrintTheory').onclick = () => {
      const workerName = requireWorkerName();
      if (!workerName || !lastInspected?.evalTheory) return;
      const { student, label, evalTheory } = lastInspected;
      const nowStr = new Date().toLocaleString('de-DE', { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' });

      const checklistHtml = [];
      for (let ch = 1; ch <= 12; ch++) {
        const cnt = evalTheory.grundCounts.get(ch) || 0;
        const isExt = evalTheory.grundExtFlags.get(ch);
        checklistHtml.push(`L${ch}: <strong>${cnt > 0 ? cnt + '×' + (isExt ? ' (Wechsler)' : '') : 'fehlt'}</strong>`);
      }
      const zEntries = Array.from(evalTheory.zusatzCounts.entries()).map(([k, c]) => `${k}: <strong>${c}×${evalTheory.zusatzExtFlags.get(k) ? ' (Wechsler)' : ''}</strong>`);
      const abnPrintInfo = evalTheory.wechslerAbnDate ? ` · ABN vom ${escapeHtml(evalTheory.wechslerAbnDate)}` : '';

      document.getElementById('printSheet').innerHTML = `
        <div class="p-hdr">
          <div>
            <div style="font-size:11px; font-weight:700; letter-spacing:1.5px; color:#00394d;">FAHRSCHULE DYNAMIC · FAHRSCHUL-MANAGER</div>
            <div style="font-size:18px; font-weight:700;">PRÜFBERICHT THEORIE-ZULASSUNG (§ 4 FahrschAusbO)</div>
          </div>
          <div style="text-align:right; font-size:11.5px;">
            <div><strong>Datum / Uhrzeit:</strong> ${nowStr} Uhr</div>
            <div><strong>Geprüft durch:</strong> ${escapeHtml(workerName)}</div>
          </div>
        </div>
        <div class="p-alert">
          PFLICHTBELEG FÜR DIE SCHÜLERAKTE: Dieser Prüfbericht muss ausgedruckt und zwingend in der physischen Schülerakte abgeheftet werden!
        </div>
        ${(evalTheory.extCount > 0 || evalTheory.wechslerAbnDate) ? `
          <div class="p-ext">
            FAHRSCHULWECHSLER: ${evalTheory.validExtCount} gültige DS${evalTheory.expiredExtCount > 0 ? ` (${evalTheory.expiredExtCount} abgelaufene DS ausgeschlossen!)` : ''} einer Fremdfahrschule (${escapeHtml(evalTheory.wechslerSchool)}${abnPrintInfo}) geprüft. Original-Ausbildungsbescheinigung (ABN) dahinter abheften!
          </div>
        ` : ''}
        <div class="p-meta">
          <div><strong>Fahrschüler:</strong> ${escapeHtml(label)}</div>
          <div><strong>Kartei-Nr.:</strong> ${escapeHtml(student?._karteiNr || '—')}</div>
          <div><strong>Klasse(n):</strong> ${escapeHtml(student?._klassenStr || '—')}</div>
          <div><strong>Ausbildungsart:</strong> ${escapeHtml(evalTheory.ruleModeLabel)}</div>
          <div><strong>Fahrschulwechsel:</strong> ${(evalTheory.extCount > 0 || evalTheory.wechslerAbnDate) ? `Ja (${evalTheory.validExtCount} gültige DS · ${escapeHtml(evalTheory.wechslerSchool)}${abnPrintInfo})` : 'Nein'}</div>
          <div><strong>Prüfer (Büro):</strong> ${escapeHtml(workerName)}</div>
        </div>
        <div class="p-verdict">
          <div style="font-size:16px; font-weight:700;">
            ERGEBNIS: ${evalTheory.allowed ? 'JA — ZUR THEORIEPRÜFUNG ZUGELASSEN' : 'NEIN — NICHT ZUR THEORIEPRÜFUNG ZUGELASSEN'}
          </div>
          <ul style="margin:6px 0 0; padding-left:18px; font-size:11px;">
            ${evalTheory.reasons.map(r => `<li>${r}</li>`).join('')}
          </ul>
        </div>
        <div style="font-size:10.5px; margin-bottom:10px; padding:6px 10px; border:1px solid #ccc;">
          <strong>Grundstoff (1–12):</strong> ${checklistHtml.join(' · ')}<br>
          <strong>Zusatzstoff (13/14):</strong> ${zEntries.length ? zEntries.join(' · ') : 'keine Einträge'}
        </div>
        <table class="p-table">
          <thead><tr><th>Nr.</th><th>Datum</th><th>Zeit</th><th>Minuten</th><th>Klasse</th><th>Kapitel</th><th>Fahrlehrer / Vor-Fahrschule</th><th>Filiale / Nachweis</th></tr></thead>
          <tbody>
            ${evalTheory.allItems.map((t, idx) => `
              <tr><td>${idx + 1}</td><td>${escapeHtml(t.datum)}</td><td>${escapeHtml(t.zeit)}</td><td>${t.min}</td><td>${escapeHtml(t.klasse || 'Grund')}</td><td>${escapeHtml(t.kapitel)}</td><td>${escapeHtml(t.lehrer)}</td><td>${escapeHtml(t.filiale)}</td></tr>
            `).join('')}
          </tbody>
        </table>
        <div class="p-sigs">
          <div class="p-sig-line"><strong>${escapeHtml(workerName)}</strong><br>Datum, Unterschrift Prüfer (Fahrschule Dynamic)</div>
          <div class="p-sig-line">In Schülerakte abgeheftet am: ______________________</div>
        </div>
      `;
      window.print();
    };

    document.getElementById('btnPrintPraxis').onclick = () => {
      const workerName = requireWorkerName();
      if (!workerName || !lastInspected?.evalPraxis) return;
      const { student, label, praxis, evalPraxis } = lastInspected;
      const nowStr = new Date().toLocaleString('de-DE', { day: '2-digit', month: '2-digit', year: 'numeric', hour: '2-digit', minute: '2-digit' });

      const abPrintStr = evalPraxis.potsdamMode
        ? `${evalPraxis.abMin}/180m (Potsdam: ohne An-/Abf.)`
        : `${evalPraxis.abMin}/180m (+ ${evalPraxis.twentyMinOnAbDate.length}×20m An-/Abf.)`;

      document.getElementById('printSheet').innerHTML = `
        <div class="p-hdr">
          <div>
            <div style="font-size:11px; font-weight:700; letter-spacing:1.5px; color:#00394d;">FAHRSCHULE DYNAMIC · FAHRSCHUL-MANAGER</div>
            <div style="font-size:18px; font-weight:700;">PRÜFBERICHT PRAXIS & PFLICHTFAHRTEN</div>
          </div>
          <div style="text-align:right; font-size:11.5px;">
            <div><strong>Datum / Uhrzeit:</strong> ${nowStr} Uhr</div>
            <div><strong>Geprüft durch:</strong> ${escapeHtml(workerName)}</div>
          </div>
        </div>
        <div class="p-alert">
          PFLICHTBELEG FÜR DIE SCHÜLERAKTE: Dieser Praxis-Prüfbericht muss ausgedruckt und zwingend in der physischen Schülerakte abgeheftet werden!
        </div>
        <div class="p-meta">
          <div><strong>Fahrschüler:</strong> ${escapeHtml(label)}</div>
          <div><strong>Kartei-Nr.:</strong> ${escapeHtml(student?._karteiNr || '—')}</div>
          <div><strong>Klasse(n):</strong> ${escapeHtml(student?._klassenStr || '—')}</div>
          <div><strong>Kategorie:</strong> ${evalPraxis.isNeuerwerber ? 'Neuerwerber (Ersterwerb)' : 'Umschreiber'} · ${evalPraxis.isB197 ? 'Mit B197' : 'Ohne B197'}${evalPraxis.potsdamMode ? ' · Filiale Potsdam' : ''}</div>
          <div><strong>Grundfahraufgaben (/Gf.):</strong> ${evalPraxis.gfLessons.length}× (${evalPraxis.gfChronologyOk ? 'Chronologie OK' : 'VERSTOSS NACH SONDERFAHRT!'})</div>
          <div><strong>Prüfer (Büro):</strong> ${escapeHtml(workerName)}</div>
        </div>
        <div class="p-verdict">
          <div style="font-size:16px; font-weight:700;">
            ERGEBNIS: ${evalPraxis.allowed ? 'JA — PFLICHTFAHRTEN & B197 VOLLSTÄNDIG KORREKT' : 'NEIN — FEHLER ODER OFFENE PFLICHTFAHRTEN'}
          </div>
          <div style="margin-top:4px; font-size:11.5px; font-weight:600;">
            Überland: ${evalPraxis.uelMin}/225m  |
            Autobahn: ${abPrintStr}  |
            Nacht: ${evalPraxis.nfMin}/135m  |
            B197: ${evalPraxis.ninetyMinDrives.length}×90m + Testfahrt: ${evalPraxis.testfahrt15.length}×
          </div>
          <ul style="margin:6px 0 0; padding-left:18px; font-size:11px;">
            ${evalPraxis.reasons.map(r => `<li>${r}</li>`).join('')}
          </ul>
        </div>
        <table class="p-table">
          <thead><tr><th>Nr.</th><th>Datum</th><th>Zeit</th><th>Min.</th><th>KFZ</th><th>Fahrlehrer</th><th>Art</th><th>Bemerkung</th><th>Inhalt (/Gf.)</th></tr></thead>
          <tbody>
            ${praxis.items.map((it, idx) => `
              <tr>
                <td>${idx + 1}</td>
                <td>${escapeHtml(it.datum)}</td>
                <td>${escapeHtml(it.zeit)}</td>
                <td>${it.min}</td>
                <td>${escapeHtml(it.kfz)}</td>
                <td>${escapeHtml(it.lehrer)}</td>
                <td>${escapeHtml(it.art)}</td>
                <td>${escapeHtml(it.bemerkung)}</td>
                <td><strong>${escapeHtml(it.inhalt)}</strong></td>
              </tr>
            `).join('')}
          </tbody>
        </table>
        <div class="p-sigs">
          <div class="p-sig-line"><strong>${escapeHtml(workerName)}</strong><br>Datum, Unterschrift Prüfer (Fahrschule Dynamic)</div>
          <div class="p-sig-line">In Schülerakte abgeheftet am: ______________________</div>
        </div>
      `;
      window.print();
    };

    // 11. Student List Filtering & Combined Inspection
    let fallbackCheckDoneForQuery = '';

    async function inspectStudentObject(s, targetTab = null, skipWorkerRequirement = false) {
      if (!s || !s.id) return;
      if (targetTab !== 'dokumente' && !skipWorkerRequirement && !requireWorkerName()) return;

      const tr = document.getElementById(`row-${s.id}`);
      if (tr) {
        document.querySelectorAll('#tbody tr.selected').forEach(el => el.classList.remove('selected'));
        tr.classList.add('selected');
        const btn = tr.querySelector('.btn-inspect');
        if (btn) { btn.disabled = true; btn.textContent = '...'; }
      }

      try {
        const [kFetched, theory, praxis] = await Promise.all([
          StudentStore.fetchKartei(s.id).catch(e => ({ error: e.message })),
          fetchTheoryOnly(s.id),
          fetchPraxisOnly(s.id)
        ]);

        renderTheoryDetails(s, `${s._nachname}, ${s._vorname}`, theory, praxis, kFetched);
        updateRowBadges(s.id);
      } catch (err) {
        console.error('[Dynamic Inspect Error]', err);
      } finally {
        if (tr) {
          const btn = tr.querySelector('.btn-inspect');
          if (btn) { btn.disabled = false; btn.textContent = 'Prüfen'; }
        }
      }

      if (targetTab) switchActiveModule(targetTab);
    }

    async function applyFilters(resetLimit = true) {
      if (resetLimit) renderLimit = 150;
      const rawText = document.getElementById('qText').value || '';
      const queryTokens = rawText.toLowerCase().trim().split(/[\s,.\-_()/]+/).filter(Boolean);
      const matchMode = document.getElementById('qMatchMode').value;
      const classQuery = (document.getElementById('qKlasse').value || '').toLowerCase().trim();

      visibleStudents = allStudents.filter(s => {
        if (selectedLetter && !s._nachname.toLowerCase().startsWith(selectedLetter)) return false;
        if (classQuery && !s._klassenLower.includes(classQuery)) return false;
        for (const tok of queryTokens) {
          if (matchMode === 'word' ? !s._words.some(w => w.startsWith(tok)) : !s._searchLower.includes(tok)) return false;
        }
        return true;
      });

      if (visibleStudents.length === 0 && rawText.trim().length >= 2 && fallbackCheckDoneForQuery !== rawText.trim()) {
        fallbackCheckDoneForQuery = rawText.trim();
        await loadStudentsFromServer(true);
        return;
      }

      const tbody = document.getElementById('tbody');
      tbody.innerHTML = '';
      const slice = visibleStudents.slice(0, renderLimit);
      const frag = document.createDocumentFragment();

      for (const s of slice) {
        const tr = document.createElement('tr');
        tr.id = `row-${s.id}`;
        if (lastInspected?.student?.id === s.id) tr.classList.add('selected');

        const cachedTheory = theoryCache.get(s.id);
        const cachedEvalT = cachedTheory ? evaluateTheoryRules(cachedTheory, s.id) : null;
        const theoryHtml = !cachedTheory
          ? `<span class="fsm-badge b-open">Ungeprüft</span>`
          : (cachedEvalT && cachedEvalT.allowed
              ? `<span class="fsm-badge b-yes">✓ THEORIE OK</span>`
              : `<span class="fsm-badge b-no">✕ THEORIE FEHLT</span>`);

        const cachedPraxis = praxisCache.get(s.id);
        const detectedCfg = autoDetectPraxisConfig(s, StudentStore.karteiMap.get(s.id), cachedPraxis);
        const cachedEvalP = cachedPraxis ? evaluatePraxisRules(cachedPraxis, detectedCfg.erwerbMode, detectedCfg.b197Mode, isPotsdamMode()) : null;
        const praxisHtml = !cachedPraxis
          ? `<span class="fsm-badge b-open">Ungeprüft</span>`
          : (cachedEvalP && cachedEvalP.allowed
              ? `<span class="fsm-badge b-yes">✓ PRAXIS OK</span>`
              : `<span class="fsm-badge b-no">✕ PRAXIS FEHLER</span>`);

        tr.innerHTML = `
          <td>${escapeHtml(s._karteiNr || '—')}</td>
          <td class="student-name-cell" style="font-weight:500;">${escapeHtml(s._nachname)}, ${escapeHtml(s._vorname)}${NotesManager.badgeHtml(s.id)}</td>
          <td>${escapeHtml(s._klassenStr || '—')}</td>
          <td class="verdict-theory-cell">${theoryHtml}</td>
          <td class="verdict-praxis-cell">${praxisHtml}</td>
          <td style="text-align:right;">
            <div style="display:inline-flex; gap:4px; align-items:center;">
              <button type="button" class="btn btn-petrol btn-sm btn-inspect" title="Theorie & Pflichtfahrten prüfen">Prüfen</button>
              <button type="button" class="btn btn-outline btn-sm btn-praxis" title="Direkt zur Pflichtfahrten-Analyse">Praxis</button>
              <button type="button" class="btn btn-outline btn-sm btn-kartei" title="Karteikarte & Büro-Notiz anzeigen">Kartei</button>
            </div>
          </td>
        `;

        tr.querySelector('.btn-inspect').onclick = (ev) => { ev.stopPropagation(); inspectStudentObject(s, null); };
        tr.querySelector('.btn-praxis').onclick = (ev) => { ev.stopPropagation(); inspectStudentObject(s, 'praxis'); };
        tr.querySelector('.btn-kartei').onclick = (ev) => { ev.stopPropagation(); inspectStudentObject(s, 'dokumente'); };
        tr.ondblclick = (ev) => { ev.stopPropagation(); inspectStudentObject(s, null); };
        frag.appendChild(tr);
      }
      tbody.appendChild(frag);

      const btnShowAll = document.getElementById('btnShowAll');
      if (visibleStudents.length > slice.length) {
        btnShowAll.style.display = 'inline-flex';
        btnShowAll.textContent = `Alle ${visibleStudents.length} Datensätze anzeigen (aktuell ${slice.length})`;
      } else {
        btnShowAll.style.display = 'none';
      }

      document.getElementById('btnBatch').disabled = visibleStudents.length === 0;
      const statusEl = document.getElementById('status');
      statusEl.style.color = 'var(--text-secondary)';
      const syncTime = StudentStore.lastFetch
        ? new Date(StudentStore.lastFetch).toLocaleTimeString('de-DE', { hour: '2-digit', minute: '2-digit' })
        : '—';
      statusEl.textContent = `Summe angezeigter Fahrschüler: ${visibleStudents.length} (von ${allStudents.length} · Stand: ${syncTime} Uhr)`;
    }

    async function loadStudentsFromServer(force = false) {
      const status = document.getElementById('status');
      status.textContent = 'Synchronisiere Fahrschüler...';
      const incInactive = document.getElementById('qActive').value === 'false';

      // Google-Sheet-Notizen im Hintergrund synchronisieren (blockiert NIEMALS das Laden der Schülerliste!)
      NotesManager.syncAll(force).then(() => {
        if (visibleStudents.length > 0) applyFilters(false);
      });

      allStudents = await StudentStore.refresh(force, incInactive);
      await applyFilters(true);
    }

    StudentStore.listeners.add((updatedList) => {
      allStudents = updatedList;
      applyFilters(false);
    });

    document.getElementById('qText').addEventListener('input', () => applyFilters(true));
    document.getElementById('qKlasse').addEventListener('input', () => applyFilters(true));
    document.getElementById('qMatchMode').addEventListener('change', () => applyFilters(true));
    document.getElementById('qRuleMode').addEventListener('change', () => {
      applyFilters(true);
      if (lastInspected) refreshCurrentEvaluation();
    });

    document.getElementById('btnShowAll').onclick = () => { renderLimit = Infinity; applyFilters(false); };
    document.getElementById('btnClearFilter').onclick = () => {
      document.getElementById('qText').value = '';
      document.getElementById('qKlasse').value = '';
      selectedLetter = '';
      alphabetBar.querySelectorAll('.char-btn').forEach((el, idx) => el.classList.toggle('selected', idx === 0));
      applyFilters(true);
    };
    document.getElementById('btnReloadList').onclick = () => loadStudentsFromServer(true);
    document.getElementById('qActive').onchange = () => loadStudentsFromServer(true);

    document.getElementById('btnBatch').onclick = async () => {
      if (!requireWorkerName()) return;
      const targets = [...visibleStudents];
      abortController = new AbortController();
      const btnBatch = document.getElementById('btnBatch');
      const btnStop = document.getElementById('btnStop');
      btnBatch.disabled = true;
      btnStop.style.display = 'inline-flex';
      btnStop.onclick = () => abortController.abort();

      let index = 0, completed = 0;
      async function worker() {
        while (index < targets.length && !abortController.signal.aborted) {
          const s = targets[index++];
          try {
            await Promise.all([
              StudentStore.fetchKartei(s.id).catch(() => null),
              fetchTheoryOnly(s.id, abortController.signal),
              fetchPraxisOnly(s.id, abortController.signal)
            ]);
            updateRowBadges(s.id);
          } catch (e) { if (e.name === 'AbortError') break; }
          completed++;
          document.getElementById('status').textContent = `Prüfe Theorie & Praxis: ${completed} / ${targets.length}`;
        }
      }
      await Promise.all(Array.from({ length: 6 }, () => worker()));
      btnBatch.disabled = false;
      btnStop.style.display = 'none';
    };

    loadStudentsFromServer(true).then(() => {
      if (initialStudentId) {
        const target = allStudents.find(s => s.id === initialStudentId);
        if (target) {
          document.getElementById('qText').value = target._karteiNr || `${target._nachname} ${target._vorname}`;
          applyFilters(true);
          inspectStudentObject(target, initialMod || null, true);
        }
      }
    });
  }
})();
