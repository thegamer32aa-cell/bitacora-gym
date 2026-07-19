/**
 * drive-auth.js
 * Módulo compartido para todas las páginas (gym, presión arterial, diario, futuras).
 * Centraliza: login OAuth2, búsqueda/creación de carpetas por ruta, subida de markdown.
 *
 * Uso en cada página:
 *   <script src="shared/drive-auth.js"></script>
 *   const drive = new DriveApp(CONFIG);
 *   drive.onStatusChange = (connected) => { ... actualizar UI ... };
 *   drive.connect();
 *   await drive.ensureFolderPath(["Carpeta", "Subcarpeta"]);
 *   await drive.uploadMarkdown(folderId, "nombre.md", contenido);
 */
class DriveApp {
  constructor(config) {
    this.config = config; // { CLIENT_ID, API_KEY, SCOPE }
    this.accessToken = null;
    this.tokenClient = null;
    this.onStatusChange = null; // callback(connected: boolean)
    this._folderCache = {}; // cache en memoria de rutas ya resueltas en esta sesión
  }

  initGoogle() {
    if (!window.google || !window.google.accounts) {
      // Google Identity Services aún no ha cargado; reintenta en breve.
      setTimeout(() => this.initGoogle(), 300);
      return;
    }
    this.tokenClient = google.accounts.oauth2.initTokenClient({
      client_id: this.config.CLIENT_ID,
      scope: this.config.SCOPE,
      callback: (resp) => {
        if (resp.error) {
          this._notify(false);
          throw new Error("Error al conectar con Google: " + resp.error);
        }
        this.accessToken = resp.access_token;
        try {
          sessionStorage.setItem("gdrive_access_token", resp.access_token);
        } catch (e) {}
        this._notify(true);
      },
    });
  }

  connect() {
    if (!this.tokenClient) {
      this.initGoogle();
      // Da tiempo a que initGoogle configure el tokenClient antes de pedir el token.
      setTimeout(() => this.tokenClient && this.tokenClient.requestAccessToken(), 400);
      return;
    }
    this.tokenClient.requestAccessToken();
  }

  isConnected() {
    return !!this.accessToken;
  }

  _notify(connected) {
    if (typeof this.onStatusChange === "function") this.onStatusChange(connected);
  }

  async _fetch(url, opts) {
    opts = opts || {};
    opts.headers = Object.assign({ Authorization: "Bearer " + this.accessToken }, opts.headers || {});
    const res = await fetch(url, opts);
    if (!res.ok) {
      const t = await res.text();
      throw new Error(`Drive API ${res.status}: ${t}`);
    }
    return res.json();
  }

  async _findOrCreateFolder(name, parentId) {
    const parentClause = parentId ? `'${parentId}' in parents` : `'root' in parents`;
    const q = encodeURIComponent(
      `name='${name.replace(/'/g, "\\'")}' and mimeType='application/vnd.google-apps.folder' and trashed=false and ${parentClause}`
    );
    const list = await this._fetch(`https://www.googleapis.com/drive/v3/files?q=${q}&fields=files(id,name)`);
    if (list.files && list.files.length) return list.files[0].id;

    const meta = { name, mimeType: "application/vnd.google-apps.folder" };
    if (parentId) meta.parents = [parentId];
    const created = await this._fetch("https://www.googleapis.com/drive/v3/files", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(meta),
    });
    return created.id;
  }

  /**
   * Busca (o crea si no existe) toda la cadena de carpetas de una ruta,
   * ej: ["02 - Cuerpo", "Presión Arterial"], y devuelve el id de la última.
   */
  async ensureFolderPath(pathArray) {
    const cacheKey = pathArray.join("/");
    if (this._folderCache[cacheKey]) return this._folderCache[cacheKey];

    let parentId = null;
    for (const segment of pathArray) {
      parentId = await this._findOrCreateFolder(segment, parentId);
    }
    this._folderCache[cacheKey] = parentId;
    return parentId;
  }

  async uploadMarkdown(folderId, filename, content) {
    const boundary = "app" + Date.now();
    const metadata = { name: filename, parents: [folderId], mimeType: "text/markdown" };
    const body =
      `--${boundary}\r\nContent-Type: application/json; charset=UTF-8\r\n\r\n${JSON.stringify(metadata)}\r\n` +
      `--${boundary}\r\nContent-Type: text/markdown\r\n\r\n${content}\r\n--${boundary}--`;
    return this._fetch("https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart", {
      method: "POST",
      headers: { "Content-Type": `multipart/related; boundary=${boundary}` },
      body,
    });
  }
}

function fechaISO() {
  const d = new Date();
  const pad = (n) => n.toString().padStart(2, "0");
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
}

function fechaLarga() {
  const d = new Date();
  const dias = ["domingo", "lunes", "martes", "miércoles", "jueves", "viernes", "sábado"];
  const meses = [
    "enero", "febrero", "marzo", "abril", "mayo", "junio",
    "julio", "agosto", "septiembre", "octubre", "noviembre", "diciembre",
  ];
  return `${dias[d.getDay()]} ${d.getDate()} de ${meses[d.getMonth()]} de ${d.getFullYear()}`;
}
