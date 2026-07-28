# CÓDIGO COMPLETO — PORTAL POP CHICLÉ PARA NETLIFY

## app/layout.tsx

```tsx
import type { Metadata } from "next";
import { Geist, Geist_Mono } from "next/font/google";
import "./globals.css";

const geistSans = Geist({
  variable: "--font-geist-sans",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

export const metadata: Metadata = {
  title: "Portal da Escola | POP CHICLÉ PSIEE",
  description:
    "Plataforma escolar para acompanhamento da assessoria pedagógica especializada POP CHICLÉ PSIEE.",
  other: {
    "codex-preview": "development",
  },
  icons: {
    icon: "/favicon.svg",
    shortcut: "/favicon.svg",
  },
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="pt-BR">
      <body
        className={`${geistSans.variable} ${geistMono.variable} antialiased`}
      >
        {children}
      </body>
    </html>
  );
}

```

## app/page.tsx

```tsx
import SchoolPortalShell from "./school-portal-shell";

export default function Home() {
  return <SchoolPortalShell />;
}

```

## app/school-portal-shell.tsx

```tsx
"use client";

import { ChangeEvent, CSSProperties, FormEvent, useCallback, useEffect, useMemo, useRef, useState } from "react";

type School = { id: string; name: string; code?: string | null };
type Access =
  | { role: "loading" }
  | { role: "guest" }
  | { role: "admin"; schools: School[] }
  | { role: "school"; school: School };
type StoredFile = { key: string; token: string; filename: string; contentType: string; size?: number };
type Metadata = { status?: string; date?: string; folder?: string; role?: string; filename?: string; contentType?: string; link?: string; token?: string; key?: string; files?: StoredFile[]; category?: string; responsible?: string; workStatus?: string; audience?: string; priority?: string; shareToken?: string };
type PortalComment = { id: number; author: string; content: string; createdAt: string };
type PortalItem = { id: number; kind: string; title: string; content: string; metadata: Metadata; createdAt: string; likes?: number; liked?: boolean; comments?: PortalComment[] };
type CommentHandler = (item: PortalItem, content: string, author: string) => void;
type Section = "inicio" | "agenda" | "calendario" | "trabalhos" | "galeria" | "materiais" | "comunicacao";
type Settings = { brandName: string; welcomeTitle: string; welcomeText: string; assessorName: string; assessorTitle: string; primary: string; pink: string; yellow: string; background: string };
const defaultSettings: Settings = { brandName: "POP CHICLÉ PSIEE", welcomeTitle: "Nossa escola em movimento.", welcomeText: "Agenda, trabalhos, registros, materiais e comunicação em um espaço exclusivo.", assessorName: "Dynjara Costa", assessorTitle: "Assessoria Pedagógica Especializada", primary: "#5b76e8", pink: "#ff4f9b", yellow: "#ffd84a", background: "#eef1f8" };

const sections: { id: Section; label: string; icon: string }[] = [
  { id: "inicio", label: "Início", icon: "⌂" },
  { id: "agenda", label: "Agenda", icon: "▣" },
  { id: "calendario", label: "Calendário", icon: "□" },
  { id: "trabalhos", label: "Trabalhos", icon: "✦" },
  { id: "galeria", label: "Galeria", icon: "▧" },
  { id: "materiais", label: "Materiais", icon: "▰" },
  { id: "comunicacao", label: "Comunicação", icon: "✉" },
];

function fileUrl(item: PortalItem, query: string, index = 0) {
  const separator = query ? "&" : "?";
  const stored = item.metadata.files?.[index];
  const tokenValue = stored?.token || item.metadata.token;
  const token = tokenValue ? `&token=${encodeURIComponent(tokenValue)}` : "";
  return `/api/portal-files${query}${separator}id=${item.id}&index=${index}${token}`;
}

function videoFileUrl(item: PortalItem, query: string, index = 0) {
  return `${fileUrl(item, query, index)}#t=0.001`;
}

function likeActor() {
  let actor = window.sessionStorage.getItem("pop_like_actor");
  if (!actor) {
    actor = crypto.randomUUID();
    window.sessionStorage.setItem("pop_like_actor", actor);
  }
  return actor;
}

async function shareItem(item: PortalItem) {
  const url = `${window.location.origin}/publicacao/${item.id}?token=${encodeURIComponent(item.metadata.shareToken || "")}&compartilhamento=${Date.now()}`;
  const data = { title: item.title, text: item.content || item.title, url };
  if (navigator.share) await navigator.share(data).catch(() => undefined);
  else {
    await navigator.clipboard.writeText(url);
    window.alert("Link da publicação copiado.");
  }
}

export default function SchoolPortalShell() {
  const [access, setAccess] = useState<Access>({ role: "loading" });
  const [selected, setSelected] = useState<School | null>(null);
  const [section, setSection] = useState<Section>("inicio");
  const [items, setItems] = useState<PortalItem[]>([]);
  const [code, setCode] = useState("");
  const [schoolName, setSchoolName] = useState("");
  const [createdCode, setCreatedCode] = useState("");
  const [codeSchoolId, setCodeSchoolId] = useState("");
  const [status, setStatus] = useState("");
  const [error, setError] = useState("");
  const [composer, setComposer] = useState(false);
  const [editing, setEditing] = useState<PortalItem | null>(null);
  const [mobileNav, setMobileNav] = useState(false);
  const [settings, setSettings] = useState<Settings>(defaultSettings);
  const [settingsOpen, setSettingsOpen] = useState(false);
  const [schoolEditor, setSchoolEditor] = useState(false);

  const school = selected ?? (access.role === "school" ? access.school : null);
  const admin = access.role === "admin";
  const query = admin && school ? `?schoolId=${encodeURIComponent(school.id)}` : "";

  const refreshAccess = useCallback(async () => {
    const response = await fetch("/api/access", { cache: "no-store" });
    setAccess(response.ok ? await response.json() : { role: "guest" });
  }, []);

  const loadItems = useCallback(async (target: School, role: Access["role"]) => {
    const params = new URLSearchParams();
    if (role === "admin") params.set("schoolId", target.id);
    else params.set("actor", likeActor());
    const suffix = params.size ? `?${params.toString()}` : "";
    for (let attempt = 0; attempt < 3; attempt++) {
      const response = await fetch(`/api/portal-items${suffix}`, { cache: "no-store" });
      const data = await response.json();
      if (response.ok) {
        setItems(data.items || []);
        return;
      }
      if (response.status !== 401 || attempt === 2) throw new Error(data.error || "Não foi possível abrir o portal.");
      await new Promise((resolve) => window.setTimeout(resolve, 250));
    }
  }, []);

  useEffect(() => {
    const activeTab = window.sessionStorage.getItem("pop_active_portal") === "1";
    fetch(activeTab ? "/api/access" : "/api/access?entry=1", { cache: "no-store" })
      .then((response) => response.ok ? response.json() : { role: "guest" })
      .then(async (nextAccess: Access) => {
        setAccess(nextAccess);
        const savedSection = window.sessionStorage.getItem("pop_portal_section");
        if (sections.some((item) => item.id === savedSection)) setSection(savedSection as Section);
        if (activeTab && nextAccess.role === "school") {
          setSelected(nextAccess.school);
          try {
            await loadItems(nextAccess.school, "school");
          } catch (reason) {
            setError(reason instanceof Error ? reason.message : "Não foi possível carregar os conteúdos.");
          }
        } else if (activeTab && nextAccess.role === "admin") {
          const savedSchoolId = window.sessionStorage.getItem("pop_selected_school");
          const savedSchool = nextAccess.schools.find((item) => item.id === savedSchoolId);
          if (savedSchool) {
            setSelected(savedSchool);
            loadItems(savedSchool, "admin").catch((reason) => setError(reason.message));
          }
        }
      })
      .catch(() => setAccess({ role: "guest" }));
    fetch("/api/settings", { cache: "no-store" }).then((response) => response.json()).then((data) => setSettings(data.settings || defaultSettings)).catch(() => undefined);
  }, [loadItems]);

  async function adminLogin(username: string, password: string) {
    const response = await fetch("/api/access", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ action: "admin-login", username, password }) });
    const data = await response.json();
    if (!response.ok) throw new Error(data.error || "Não foi possível entrar.");
    window.sessionStorage.setItem("pop_active_portal", "1");
    await refreshAccess();
  }

  async function saveSettings(next: Settings) {
    const response = await fetch("/api/settings", { method: "PATCH", headers: { "content-type": "application/json" }, body: JSON.stringify(next) });
    if (!response.ok) return setStatus("Não foi possível salvar a aparência.");
    setSettings(next); setSettingsOpen(false); setStatus("Aparência e textos atualizados ✓");
    window.setTimeout(() => setStatus(""), 2200);
  }

  useEffect(() => {
    if (selected) window.sessionStorage.setItem("pop_portal_section", section);
  }, [section, selected]);

  const publicItems = useMemo(
    () => items.filter((item) => admin || item.metadata.status !== "pendente"),
    [admin, items],
  );

  const byKind = useCallback((kind: string) => publicItems.filter((item) => item.kind === kind), [publicItems]);

  async function login(event: FormEvent) {
    event.preventDefault();
    setError("");
    const response = await fetch("/api/access", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ action: "login", code }),
    });
    const data = await response.json();
    if (!response.ok) return setError(data.error || "Código inválido.");
    window.sessionStorage.setItem("pop_active_portal", "1");
    setAccess({ role: "school", school: data.school });
    setSelected(data.school);
    try {
      await loadItems(data.school, "school");
    } catch (reason) {
      setError(reason instanceof Error ? reason.message : "Não foi possível carregar os conteúdos.");
    }
    setCode("");
  }

  async function createSchool(event: FormEvent) {
    event.preventDefault();
    if (!schoolName.trim()) return;
    const response = await fetch("/api/access", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ action: "create-school", name: schoolName }),
    });
    const data = await response.json();
    if (!response.ok) return setError(data.error || "Não foi possível criar a escola.");
    setCreatedCode(data.code);
    setCodeSchoolId(data.school.id);
    setSchoolName("");
    await refreshAccess();
  }

  async function resetCode(target: School) {
    const response = await fetch("/api/access", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ action: "reset-code", schoolId: target.id }),
    });
    const data = await response.json();
    if (response.ok) {
      setCreatedCode(data.code);
      setCodeSchoolId(target.id);
      await refreshAccess();
    }
  }

  async function openSchool(target: School) {
    setSelected(target);
    setSection("inicio");
    window.sessionStorage.setItem("pop_selected_school", target.id);
    window.sessionStorage.setItem("pop_portal_section", "inicio");
    setStatus("Abrindo o portal...");
    try {
      await loadItems(target, access.role);
      setStatus("");
    } catch (reason) {
      setError(reason instanceof Error ? reason.message : "Não foi possível abrir o portal.");
      setSelected(null);
    }
  }

  async function logout() {
    await fetch("/api/access", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ action: "logout" }),
    });
    setSelected(null);
    setItems([]);
    setAccess({ role: "guest" });
    window.sessionStorage.removeItem("pop_active_portal");
    window.sessionStorage.removeItem("pop_selected_school");
    window.sessionStorage.removeItem("pop_portal_section");
  }

  async function saveTextItem(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    if (!school) return;
    const data = new FormData(event.currentTarget);
    const payload = {
      id: editing?.id,
      kind: String(data.get("kind") || ""),
      title: String(data.get("title") || ""),
      content: String(data.get("content") || ""),
      metadata: {
        date: String(data.get("date") || ""),
        role: String(data.get("role") || ""),
        link: String(data.get("link") || ""),
        audience: String(data.get("audience") || ""),
        priority: String(data.get("priority") || ""),
        status: String(data.get("status") || (admin ? "publicado" : "pendente")),
      },
    };
    setStatus("Salvando...");
    const response = await fetch(`/api/portal-items${query}`, {
      method: editing ? "PATCH" : "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(payload),
    });
    const result = await response.json();
    if (!response.ok) return setStatus(result.error || "Não foi possível salvar.");
    await loadItems(school, access.role);
    setComposer(false);
    setEditing(null);
    setStatus(admin ? "Publicação salva ✓" : "Recado enviado para aprovação ✓");
    window.setTimeout(() => setStatus(""), 2200);
  }

  async function saveFile(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    if (!school) return;
    await uploadFormData(new FormData(event.currentTarget));
  }

  async function uploadFormData(body: FormData, successMessage = "Arquivo publicado ✓", closeComposer = true) {
    if (!school) return;
    const uploadFiles = [...body.getAll("files"), body.get("file")].filter((entry): entry is File => entry instanceof File && entry.size > 0);
    if (!uploadFiles.length) return setStatus("Selecione pelo menos um arquivo.");
    if (uploadFiles.length > 10) return setStatus("Envie no máximo 10 arquivos por publicação.");
    if (uploadFiles.reduce((total, file) => total + file.size, 0) > 1024 * 1024 * 1024) return setStatus("A publicação pode ter até 1 GB no total.");
    setStatus("Enviando arquivo...");
    try {
      if (uploadFiles.length) {
        const files: StoredFile[] = [];
        for (let fileIndex = 0; fileIndex < uploadFiles.length; fileIndex++) {
          const file = uploadFiles[fileIndex];
          setStatus(`Enviando ${file.name} (${fileIndex + 1}/${uploadFiles.length})...`);
          const initResponse = await fetch(`/api/portal-upload${query}`, {
            method: "POST", headers: { "content-type": "application/json" },
            body: JSON.stringify({ action: "init", filename: file.name, contentType: file.type || "application/octet-stream" }),
          });
          const init = await initResponse.json();
          if (!initResponse.ok) throw new Error(init.error || "Não foi possível iniciar o vídeo.");
          const uploadResponse = await fetch(init.signedUrl, {
            method: "PUT",
            headers: { "content-type": file.type || "application/octet-stream", "x-upsert": "false" },
            body: file,
          });
          if (!uploadResponse.ok) throw new Error(`O armazenamento recusou ${file.name}. Confira o limite configurado no Supabase.`);
          setStatus(`Enviando ${file.name}: 100%`);
          files.push({ key: init.key, token: init.token, filename: file.name, contentType: file.type || "application/octet-stream", size: file.size });
        }
        const publishPayload: Record<string, unknown> = { action: "publish", files };
        for (const key of ["kind", "title", "content", "folder", "date", "category", "responsible", "link", "workStatus", "audience", "priority"]) {
          publishPayload[key] = String(body.get(key) || "");
        }
        const publishResponse = await fetch(`/api/portal-upload${query}`, {
          method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify(publishPayload),
        });
        const published = await publishResponse.json();
        if (!publishResponse.ok) throw new Error(published.error || "Não foi possível publicar o vídeo.");
        await loadItems(school, access.role);
        if (closeComposer) setComposer(false);
        setStatus(successMessage);
        window.setTimeout(() => setStatus(""), 2200);
        return;
      }
    } catch (reason) {
      setStatus(`O envio não foi concluído: ${reason instanceof Error ? reason.message : "erro de conexão"}.`);
    }
  }

  async function saveWork(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    if (!school) return;
    const body = new FormData(event.currentTarget);
    const files = body.getAll("files").filter((entry) => entry instanceof File && entry.size > 0);
    setStatus(editing ? "Salvando alterações..." : "Publicando trabalho...");
    let response: Response;
    if (editing) {
      response = await fetch(`/api/portal-items${query}`, {
        method: "PATCH", headers: { "content-type": "application/json" },
        body: JSON.stringify({
          id: editing.id, title: body.get("title"), content: body.get("content"),
          metadata: { date: body.get("date"), category: body.get("category"), responsible: body.get("responsible"), link: body.get("link"), workStatus: body.get("workStatus") },
        }),
      });
    } else if (files.length > 0) {
      await uploadFormData(body, "Trabalho salvo ✓");
      return;
    } else {
      response = await fetch(`/api/portal-items${query}`, {
        method: "POST", headers: { "content-type": "application/json" },
        body: JSON.stringify({
          kind: "trabalho", title: body.get("title"), content: body.get("content"),
          metadata: { date: body.get("date"), category: body.get("category"), responsible: body.get("responsible"), link: body.get("link"), workStatus: body.get("workStatus") },
        }),
      });
    }
    const result = await response.json();
    if (!response.ok) return setStatus(result.error || "Não foi possível salvar o trabalho.");
    await loadItems(school, access.role);
    setComposer(false); setEditing(null); setStatus("Trabalho salvo ✓");
    window.setTimeout(() => setStatus(""), 2200);
  }

  async function saveCommunication(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    if (!school) return;
    const body = new FormData(event.currentTarget);
    const files = body.getAll("files").filter((entry) => entry instanceof File && entry.size > 0);
    if (files.length) return saveFile(event);
    return saveTextItem(event);
  }

  async function updateSchoolProfile(name: string, subtitle: string) {
    if (!school) return;
    const response = await fetch("/api/access", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ action: "update-school", schoolId: school.id, name, subtitle }) });
    const data = await response.json();
    if (!response.ok) return setStatus(data.error || "Não foi possível editar a escola.");
    setSelected(data.school);
    await loadItems(data.school, access.role);
    setSchoolEditor(false); setStatus("Perfil da escola atualizado ✓");
  }

  async function removeItem(item: PortalItem) {
    if (!school || !window.confirm(`Excluir “${item.title}”?`)) return;
    const response = await fetch(`/api/portal-items${query}${query ? "&" : "?"}id=${item.id}`, { method: "DELETE" });
    if (response.ok) await loadItems(school, access.role);
  }

  async function changeSchoolPhoto(event: ChangeEvent<HTMLInputElement>) {
    const file = event.target.files?.[0];
    if (!file || !school) return;
    const body = new FormData();
    body.set("kind", "perfil");
    body.set("title", `Foto de ${school.name}`);
    body.set("file", file);
    setStatus("Enviando a foto da escola...");
    try {
      await uploadFormData(body, "Foto da escola atualizada ✓", false);
      event.target.value = "";
    } catch {
      setStatus("A foto não foi enviada. Use JPG ou PNG com até 20 MB.");
    }
  }

  async function approve(item: PortalItem) {
    if (!school) return;
    await fetch(`/api/portal-items${query}`, {
      method: "PATCH",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ id: item.id, metadata: { status: "publicado" } }),
    });
    await loadItems(school, access.role);
  }

  async function toggleLike(item: PortalItem) {
    if (!school) return;
    await fetch(`/api/portal-items${query}`, {
      method: "POST", headers: { "content-type": "application/json" },
      body: JSON.stringify({ action: "toggle-like", parentId: item.id, actor: admin ? "admin" : likeActor() }),
    });
    await loadItems(school, access.role);
  }

  async function addComment(item: PortalItem, content: string, author: string) {
    if (!school || !content.trim() || !author.trim()) return;
    const response = await fetch(`/api/portal-items${query}`, {
      method: "POST", headers: { "content-type": "application/json" },
      body: JSON.stringify({ kind: "comentario", title: author.trim(), content, metadata: { parentId: item.id } }),
    });
    if (response.ok) await loadItems(school, access.role);
  }

  if (access.role === "loading") return <Loading />;
  const themeStyle = { "--blue": settings.primary, "--pink": settings.pink, "--yellow": settings.yellow, "--bg": settings.background } as CSSProperties;
  if (access.role === "guest") return <div style={themeStyle}><Login code={code} setCode={setCode} error={error} settings={settings} onSubmit={login} onAdminLogin={adminLogin} /></div>;
  if (admin && !selected) {
    return (
      <div style={themeStyle}><AdminSchools
        access={access}
        schoolName={schoolName}
        setSchoolName={setSchoolName}
        createdCode={createdCode}
        codeSchoolId={codeSchoolId}
        onCreate={createSchool}
        onOpen={openSchool}
        onReset={resetCode}
        onShowCode={(target) => { setCodeSchoolId(target.id); setCreatedCode(target.code || ""); }}
        settings={settings}
        onEditSettings={() => setSettingsOpen(true)}
        onLogout={logout}
      />{settingsOpen ? <SettingsEditor settings={settings} onClose={() => setSettingsOpen(false)} onSave={saveSettings} /> : null}</div>
    );
  }

  return (
    <main className="orkut-shell" style={themeStyle}>
      <header className="orkut-topbar">
        <button className="mobile-menu" onClick={() => setMobileNav(!mobileNav)} aria-label="Abrir menu">☰</button>
        <div className="brand-word">{settings.brandName}</div>
        <div className="top-school"><strong>{school?.name}</strong><span>{admin ? "modo administradora" : "portal exclusivo da escola"}</span></div>
        <div className="top-actions">
          {admin ? <button onClick={() => { setSelected(null); setItems([]); window.sessionStorage.removeItem("pop_selected_school"); window.sessionStorage.removeItem("pop_portal_section"); }}>Trocar de escola</button> : null}
          {admin ? <button onClick={() => setSettingsOpen(true)}>✎ Editar portal</button> : null}
          <button className="top-logout" onClick={logout}>Sair</button>
        </div>
      </header>

      <div className="orkut-layout">
        <aside className={`left-column${mobileNav ? " open" : ""}`}>
          <div className="school-profile">
            <div className="school-avatar">{byKind("perfil")[0] ? <img src={fileUrl(byKind("perfil")[0], query)} alt={`Foto de ${school?.name}`} /> : <span>🏫</span>}</div>
            <h1>{school?.name}</h1>
            <p>{byKind("config")[0]?.content || "Escola assessorada pela POP CHICLÉ PSIEE"}</p>
            {admin ? <><label className="change-photo">Trocar foto<input type="file" accept="image/*" onChange={changeSchoolPhoto} /></label>{byKind("perfil")[0] ? <button className="delete-photo" onClick={() => removeItem(byKind("perfil")[0])}>Excluir foto</button> : null}</> : null}
            {admin ? <button className="edit-school-profile" onClick={() => setSchoolEditor(true)}>Editar nome e descrição</button> : null}
          </div>
          <nav className="orkut-nav">
            {sections.map((item) => (
              <button key={item.id} className={section === item.id ? "active" : ""} onClick={() => { setSection(item.id); setMobileNav(false); }}>
                <span>{item.icon}</span>{item.label}
              </button>
            ))}
            <button className="mobile-logout" onClick={logout}><span>↪</span>Sair</button>
          </nav>
          <div className="access-note"><b>🔒 Espaço exclusivo</b><p>Os conteúdos desta escola não aparecem nos portais das demais.</p></div>
        </aside>

        <section className="center-column">
          <SectionHeader section={section} admin={admin} onAdd={() => { setEditing(null); setComposer(true); }} />
          {section === "inicio" ? <Home school={school} items={publicItems} admin={admin} onNavigate={setSection} onLike={toggleLike} onComment={addComment} onEdit={(item) => { setSection(item.kind === "trabalho" ? "trabalhos" : item.kind === "galeria" ? "galeria" : item.kind === "agenda" ? "agenda" : "comunicacao"); setEditing(item); setComposer(true); }} onDelete={removeItem} /> : null}
          {section === "agenda" ? <Agenda items={byKind("agenda")} admin={admin} onEdit={(item) => { setEditing(item); setComposer(true); }} onDelete={removeItem} /> : null}
          {section === "calendario" ? <Calendar items={byKind("agenda")} /> : null}
          {section === "trabalhos" ? <Feed items={byKind("trabalho")} empty="Nenhum trabalho publicado ainda." admin={admin} onEdit={(item) => { setEditing(item); setComposer(true); }} onDelete={removeItem} onLike={toggleLike} onComment={addComment} /> : null}
          {section === "galeria" ? <Gallery items={byKind("galeria")} admin={admin} onEdit={(item) => { setEditing(item); setComposer(true); }} onDelete={removeItem} query={query} onLike={toggleLike} onComment={addComment} /> : null}
          {section === "materiais" ? <Materials folders={byKind("pasta")} items={byKind("material")} admin={admin} onEdit={(item) => { setEditing(item); setComposer(true); }} onDelete={removeItem} query={query} /> : null}
          {section === "comunicacao" ? <Communication items={[...byKind("orientacao"), ...byKind("recado"), ...byKind("depoimento")].sort((a, b) => b.id - a.id)} admin={admin} onApprove={approve} onDelete={removeItem} onMessage={() => { setEditing(null); setComposer(true); }} onLike={toggleLike} onComment={addComment} /> : null}
        </section>

        <aside className="right-column">
          <div className="advisor-card">
            <div className="advisor-avatar">DC</div>
            <h2>{settings.assessorName}</h2>
            <p>{settings.assessorTitle}</p>
            <span className="online">● acompanhamento ativo</span>
          </div>
          <div className="summary-card">
            <h3>Movimento da escola</h3>
            <div><b>{byKind("agenda").length}</b><span>compromissos</span></div>
            <div><b>{byKind("trabalho").length}</b><span>trabalhos</span></div>
            <div><b>{byKind("material").length}</b><span>materiais</span></div>
          </div>
          <div className="method-card"><b>Conhecimentos que grudam.</b><p>Estratégias que flexionam.</p></div>
        </aside>
      </div>

      {composer && school ? (
        <Composer
          section={section}
          admin={admin}
          editing={editing}
          onClose={() => { setComposer(false); setEditing(null); }}
          onSaveText={saveTextItem}
          onSaveFile={saveFile}
          onSaveWork={saveWork}
          onSaveCommunication={saveCommunication}
        />
      ) : null}
      {status ? <div className="portal-toast">{status}</div> : null}
      {settingsOpen ? <SettingsEditor settings={settings} onClose={() => setSettingsOpen(false)} onSave={saveSettings} /> : null}
      {schoolEditor && school ? <SchoolEditor school={school} subtitle={byKind("config")[0]?.content || "Escola assessorada pela POP CHICLÉ PSIEE"} onClose={() => setSchoolEditor(false)} onSave={updateSchoolProfile} /> : null}
    </main>
  );
}

function Loading() {
  return <main className="auth-page"><div className="auth-card"><b>POP CHICLÉ PSIEE</b><p>Abrindo a plataforma...</p></div></main>;
}

function Login({ code, setCode, error, settings, onSubmit, onAdminLogin }: { code: string; setCode: (value: string) => void; error: string; settings: Settings; onSubmit: (event: FormEvent) => void; onAdminLogin: (username: string, password: string) => Promise<void> }) {
  const [adminMode, setAdminMode] = useState(false);
  const [adminError, setAdminError] = useState("");
  return (
    <main className="auth-page">
      <section className="auth-card">
        <div className="auth-logo"><b>{settings.brandName}</b></div>
        <p className="eyebrow">PORTAL DA ESCOLA ASSESSORADA</p>
        <h1>{settings.welcomeTitle}</h1>
        <p>{settings.welcomeText}</p>
        {!adminMode ? <form onSubmit={onSubmit}>
          <label>Código exclusivo da escola<input value={code} onChange={(event) => setCode(event.target.value.toUpperCase())} placeholder="POP-XXXXXXX" /></label>
          <button>Entrar no portal</button>
        </form> : <form onSubmit={async (event) => { event.preventDefault(); const data = new FormData(event.currentTarget); try { await onAdminLogin(String(data.get("username")), String(data.get("password"))); } catch (reason) { setAdminError(reason instanceof Error ? reason.message : "Não foi possível entrar."); } }}>
          <label>Usuário administrativo<input name="username" autoComplete="username" required /></label>
          <label>Senha<input name="password" type="password" autoComplete="current-password" required /></label>
          <button>Entrar como administradora</button>
        </form>}
        {error || adminError ? <p className="form-error">{adminError || error}</p> : null}
        <button className="switch-login" onClick={() => { setAdminMode(!adminMode); setAdminError(""); }}>{adminMode ? "Voltar ao acesso da escola" : "Acesso da administradora"}</button>
      </section>
    </main>
  );
}

function AdminSchools({ access, schoolName, setSchoolName, createdCode, codeSchoolId, onCreate, onOpen, onReset, onShowCode, settings, onEditSettings, onLogout }: {
  access: Extract<Access, { role: "admin" }>; schoolName: string; setSchoolName: (value: string) => void; createdCode: string; codeSchoolId: string;
  onCreate: (event: FormEvent) => void; onOpen: (school: School) => void; onReset: (school: School) => void; onShowCode: (school: School) => void; settings: Settings; onEditSettings: () => void; onLogout: () => void;
}) {
  const codeSchool = access.schools.find((school) => school.id === codeSchoolId);
  return (
    <main className="admin-home">
      <header><div className="brand-word">{settings.brandName}</div><div className="admin-header-actions"><div><strong>{settings.assessorName}</strong><span>Administração</span></div><button onClick={onLogout}>Sair</button></div></header>
      <div className="admin-orkut-layout"><aside className="admin-profile"><div className="advisor-avatar">DC</div><h2>{settings.assessorName}</h2><p>{settings.assessorTitle}</p><button onClick={onEditSettings}>✎ Editar aparência e textos</button><nav><b>⌂ Início</b><span>▣ Escolas</span><span>✉ Publicações</span><span>▧ Arquivos</span><span>⚙ Configurações</span></nav></aside><section className="admin-center">
      <article className="admin-welcome"><small>INÍCIO &gt; PERFIL DA ASSESSORIA</small><h1>Olá, {settings.assessorName}! 😊</h1><p>Gerencie sua rede, abra o perfil de cada escola e edite todos os conteúdos da assessoria.</p><div><span><b>{access.schools.length}</b> escolas</span><span><b>ADM</b> acesso total</span><span><b>24h</b> portal ativo</span></div></article>
      <section className="create-school">
        <form onSubmit={onCreate}><label>Nome da escola<input value={schoolName} onChange={(event) => setSchoolName(event.target.value)} placeholder="Ex.: Escola Cassiano" /></label><button>Criar portal</button></form>
        {codeSchool ? <div className="created-code"><span>CÓDIGO DE {codeSchool.name}</span>{createdCode ? <><strong>{createdCode}</strong><button onClick={() => navigator.clipboard.writeText(createdCode)}>Copiar código</button></> : <small>Código anterior protegido. Use “Novo código” apenas uma vez para ele ficar disponível aqui.</small>}</div> : null}
      </section>
      <section className="school-grid">
        {access.schools.map((school) => <article key={school.id} onClick={() => onShowCode(school)}><div className="folder-icon">▰</div><div><small>ESCOLA ASSESSORADA</small><h2>{school.name}</h2><p>Agenda, trabalhos, materiais e comunicação exclusivos.</p></div><button onClick={(event) => { event.stopPropagation(); onOpen(school); }}>Abrir portal</button><button className="secondary" onClick={(event) => { event.stopPropagation(); onReset(school); }}>Novo código</button></article>)}
        {!access.schools.length ? <div className="empty-state">Cadastre a primeira escola para começar.</div> : null}
      </section></section><aside className="admin-right"><h3>Minha rede</h3><p><b>{access.schools.length}</b> escolas assessoradas</p><p>Entre em uma escola para publicar, anexar, editar ou excluir.</p><button onClick={onEditSettings}>Personalizar portal</button></aside></div>
    </main>
  );
}

function SettingsEditor({ settings, onClose, onSave }: { settings: Settings; onClose: () => void; onSave: (settings: Settings) => void }) {
  return <div className="modal-backdrop"><section className="composer settings-editor"><header><div><span>MODO DE EDIÇÃO GERAL</span><h2>Personalizar portal</h2></div><button onClick={onClose}>×</button></header><form onSubmit={(event) => { event.preventDefault(); const data = new FormData(event.currentTarget); onSave(Object.fromEntries(data) as unknown as Settings); }}><label>Nome da marca<input name="brandName" defaultValue={settings.brandName} /></label><label>Frase principal<input name="welcomeTitle" defaultValue={settings.welcomeTitle} /></label><label>Texto de apresentação<textarea name="welcomeText" rows={3} defaultValue={settings.welcomeText} /></label><div className="form-row"><label>Nome da assessora<input name="assessorName" defaultValue={settings.assessorName} /></label><label>Cargo/apresentação<input name="assessorTitle" defaultValue={settings.assessorTitle} /></label></div><div className="color-grid"><label>Cor principal<input name="primary" type="color" defaultValue={settings.primary} /></label><label>Rosa<input name="pink" type="color" defaultValue={settings.pink} /></label><label>Amarelo<input name="yellow" type="color" defaultValue={settings.yellow} /></label><label>Fundo<input name="background" type="color" defaultValue={settings.background} /></label></div><button>Salvar tudo</button></form></section></div>;
}

function SchoolEditor({ school, subtitle, onClose, onSave }: { school: School; subtitle: string; onClose: () => void; onSave: (name: string, subtitle: string) => void }) {
  return <div className="modal-backdrop"><section className="composer"><header><div><span>EDITAR PERFIL</span><h2>Dados da escola</h2></div><button onClick={onClose}>×</button></header><form onSubmit={(event) => { event.preventDefault(); const data = new FormData(event.currentTarget); onSave(String(data.get("name")), String(data.get("subtitle"))); }}><label>Nome da escola<input name="name" defaultValue={school.name} required /></label><label>Descrição abaixo do nome<textarea name="subtitle" rows={3} defaultValue={subtitle} required /></label><button>Salvar perfil da escola</button></form></section></div>;
}

function SectionHeader({ section, admin, onAdd }: { section: Section; admin: boolean; onAdd: () => void }) {
  const copy: Record<Section, [string, string]> = {
    inicio: ["Início", "Visão geral do acompanhamento"],
    agenda: ["Agenda", "Compromissos e prazos da assessoria"],
    calendario: ["Calendário", "Organização mensal dos compromissos"],
    trabalhos: ["Trabalhos", "Registros do que estamos construindo juntos"],
    galeria: ["Galeria", "Imagens que contam o percurso da escola"],
    materiais: ["Materiais", "Pastas e arquivos exclusivos da escola"],
    comunicacao: ["Comunicação", "Orientações, recados e devolutivas"],
  };
  const canAdd = admin && !["inicio", "calendario"].includes(section);
  return <header className="section-head"><div><span>{copy[section][1]}</span><h2>{copy[section][0]}</h2></div>{canAdd ? <button onClick={onAdd}>＋ Publicar</button> : null}</header>;
}

function Home({ school, items, admin, onNavigate, onLike, onComment, onEdit, onDelete }: { school: School | null; items: PortalItem[]; admin: boolean; onNavigate: (section: Section) => void; onLike: (item: PortalItem) => void; onComment: CommentHandler; onEdit: (item: PortalItem) => void; onDelete: (item: PortalItem) => void }) {
  const recent = items.filter((item) => ["agenda", "trabalho", "galeria", "orientacao", "recado", "depoimento"].includes(item.kind)).slice(0, 3);
  return (
    <>
      <article className="orkut-home-card"><header>Início &gt; Perfil da escola</header><div><p className="orkut-hello">Olá, {school?.name}!</p><p>Este é o espaço da sua escola na rede POP CHICLÉ PSIEE. Aqui ficam o mural, os recados, as fotos e todo o percurso da assessoria.</p><section><span><b>{items.length}</b> atualizações</span><span><b>{items.reduce((total, item) => total + (item.likes || 0), 0)}</b> curtidas</span><span><b>{items.reduce((total, item) => total + (item.comments?.length || 0), 0)}</b> comentários</span></section></div></article>
      <div className="shortcut-grid">
        {sections.slice(1, 7).map((item) => <button key={item.id} onClick={() => onNavigate(item.id)}><span>{item.icon}</span><b>{item.label}</b><small>Abrir espaço</small></button>)}
      </div>
      <div className="feed-title"><h3>Atualizações recentes</h3></div>
      {recent.length ? <Feed items={recent} empty="" admin={admin} onEdit={onEdit} onDelete={onDelete} onLike={onLike} onComment={onComment} /> : <div className="empty-state">As primeiras atualizações da assessoria aparecerão aqui.</div>}
    </>
  );
}

function ItemActions({ item, onEdit, onDelete }: { item: PortalItem; onEdit: (item: PortalItem) => void; onDelete: (item: PortalItem) => void }) {
  return <div className="item-actions"><button onClick={() => onEdit(item)}>Editar</button><button onClick={() => onDelete(item)}>Excluir</button></div>;
}

function Feed({ items, empty, admin, onEdit, onDelete, onLike, onComment }: { items: PortalItem[]; empty: string; admin: boolean; onEdit: (item: PortalItem) => void; onDelete: (item: PortalItem) => void; onLike: (item: PortalItem) => void; onComment: CommentHandler }) {
  if (!items.length) return <div className="empty-state">{empty}</div>;
  return <div className="feed">{items.map((item) => <article className="post-card" key={item.id}><div className="post-avatar">PC</div><div className="post-body"><header><b>POP CHICLÉ PSIEE</b><span>{new Date(item.createdAt).toLocaleDateString("pt-BR")}</span></header><h3>{item.title}</h3>{item.kind === "trabalho" ? <WorkDetails item={item} /> : <p>{item.content}</p>}{admin ? <ItemActions item={item} onEdit={onEdit} onDelete={onDelete} /> : null}<Engagement item={item} onLike={onLike} onComment={onComment} /></div></article>)}</div>;
}

function WorkDetails({ item }: { item: PortalItem }) {
  return <div className="work-details">{item.content ? <p>{item.content}</p> : null}<div className="work-meta">{item.metadata.workStatus ? <span>{item.metadata.workStatus}</span> : null}{item.metadata.category ? <span>{item.metadata.category}</span> : null}{item.metadata.date ? <span>📅 {new Date(`${item.metadata.date}T12:00:00`).toLocaleDateString("pt-BR")}</span> : null}{item.metadata.responsible ? <span>👤 {item.metadata.responsible}</span> : null}</div><MediaCarousel item={item} />{item.metadata.link ? <a className="work-file" href={item.metadata.link} target="_blank" rel="noreferrer">🔗 Abrir link relacionado</a> : null}</div>;
}

function MediaCarousel({ item }: { item: PortalItem }) {
  const files = item.metadata.files || (item.metadata.key ? [{ key: item.metadata.key, token: item.metadata.token || "", filename: item.metadata.filename || "arquivo", contentType: item.metadata.contentType || "" }] : []);
  const [index, setIndex] = useState(0);
  if (!files.length) return null;
  const safeIndex = Math.min(index, files.length - 1);
  const file = files[safeIndex];
  const url = fileUrl(item, "", safeIndex);
  const pdf = file.contentType === "application/pdf" || file.filename.toLowerCase().endsWith(".pdf");
  return <div className="portal-media-carousel">{file.contentType.startsWith("image/") ? <img src={url} alt={file.filename} /> : file.contentType.startsWith("video/") ? <video src={videoFileUrl(item, "", safeIndex)} controls playsInline preload="auto" /> : pdf ? <><PdfThumbnail url={url} filename={file.filename} /><a className="media-open" href={url} target="_blank" rel="noreferrer">Abrir PDF</a></> : <a className="gallery-file" href={url} target="_blank" rel="noreferrer">📎<b>{file.filename}</b><span>Abrir arquivo</span></a>}{files.length > 1 ? <><button className="carousel-prev" type="button" onClick={() => setIndex((safeIndex - 1 + files.length) % files.length)} aria-label="Arquivo anterior">‹</button><button className="carousel-next" type="button" onClick={() => setIndex((safeIndex + 1) % files.length)} aria-label="Próximo arquivo">›</button><div className="carousel-count">{safeIndex + 1}/{files.length}</div></> : null}</div>;
}

function PdfThumbnail({ url, filename }: { url: string; filename: string }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [failed, setFailed] = useState(false);
  useEffect(() => {
    let cancelled = false;
    let destroy: (() => void) | undefined;
    import("pdfjs-dist").then(async (pdfjs) => {
      if (cancelled) return;
      pdfjs.GlobalWorkerOptions.workerSrc = new URL(
        "pdfjs-dist/build/pdf.worker.min.mjs",
        import.meta.url,
      ).toString();
      const task = pdfjs.getDocument({ url, isEvalSupported: false });
      destroy = () => { void task.destroy(); };
      const document = await task.promise;
      const page = await document.getPage(1);
      const original = page.getViewport({ scale: 1 });
      const scale = Math.min(1200 / original.width, 900 / original.height);
      const viewport = page.getViewport({ scale });
      const canvas = canvasRef.current;
      const context = canvas?.getContext("2d");
      if (!canvas || !context || cancelled) return;
      canvas.width = Math.floor(viewport.width);
      canvas.height = Math.floor(viewport.height);
      context.fillStyle = "#fff";
      context.fillRect(0, 0, canvas.width, canvas.height);
      await page.render({ canvas, canvasContext: context, viewport }).promise;
    }).catch(() => {
      if (!cancelled) setFailed(true);
    });
    return () => {
      cancelled = true;
      destroy?.();
    };
  }, [url]);
  return failed ? <div className="pdf-fallback">PDF<br /><small>{filename}</small></div> : <canvas ref={canvasRef} aria-label={`Primeira página de ${filename}`} />;
}

function Engagement({ item, onLike, onComment }: { item: PortalItem; onLike: (item: PortalItem) => void; onComment: CommentHandler }) {
  const [text, setText] = useState("");
  const [author, setAuthor] = useState("");
  const [commentsOpen, setCommentsOpen] = useState(true);
  useEffect(() => {
    setAuthor(window.sessionStorage.getItem("pop_comment_author") || "");
  }, []);
  const commentCount = item.comments?.length || 0;
  return <div className="engagement"><div className="social-actions"><button className={item.liked ? "liked" : ""} onClick={() => onLike(item)}>♥ {item.liked ? "Curtido" : "Curtir"} <span>{item.likes || 0}</span></button><button onClick={() => shareItem(item)}>➤ Compartilhar</button>{commentCount ? <button onClick={() => setCommentsOpen(!commentsOpen)} aria-expanded={commentsOpen}>{commentsOpen ? "▾ Ocultar" : "▸ Ver"} {commentCount} comentário{commentCount === 1 ? "" : "s"}</button> : null}</div>{commentCount && commentsOpen ? <div className="comment-list">{item.comments?.map((comment) => <p key={comment.id}><b>{comment.author}</b>{comment.content}</p>)}</div> : null}<form onSubmit={(event) => { event.preventDefault(); if (text.trim() && author.trim()) { window.sessionStorage.setItem("pop_comment_author", author.trim()); onComment(item, text, author); setText(""); } }}><input className="comment-author" value={author} onChange={(event) => setAuthor(event.target.value)} placeholder="Seu nome" aria-label="Nome de quem comenta" required /><input value={text} onChange={(event) => setText(event.target.value)} placeholder="Escreva um comentário..." aria-label="Comentário" required /><button>Comentar</button></form></div>;
}

function Agenda({ items, admin, onEdit, onDelete }: { items: PortalItem[]; admin: boolean; onEdit: (item: PortalItem) => void; onDelete: (item: PortalItem) => void }) {
  if (!items.length) return <div className="empty-state">Nenhum compromisso publicado ainda.</div>;
  return <div className="agenda-list">{items.map((item) => <article key={item.id} id={`post-${item.id}`}><time>{item.metadata.date ? new Date(`${item.metadata.date}T12:00:00`).toLocaleDateString("pt-BR", { day: "2-digit", month: "short" }) : "A definir"}</time><div><h3>{item.title}</h3><p>{item.content}</p><button className="share-button" onClick={() => shareItem(item)}>➤ Compartilhar</button>{admin ? <ItemActions item={item} onEdit={onEdit} onDelete={onDelete} /> : null}</div></article>)}</div>;
}

function Calendar({ items }: { items: PortalItem[] }) {
  const [month, setMonth] = useState(() => new Date(new Date().getFullYear(), new Date().getMonth(), 1));
  const days = new Date(month.getFullYear(), month.getMonth() + 1, 0).getDate();
  const first = new Date(month.getFullYear(), month.getMonth(), 1).getDay();
  const dated = new Map(items.map((item) => [item.metadata.date, item]));
  return <div className="calendar-card"><header><button onClick={() => setMonth(new Date(month.getFullYear(), month.getMonth() - 1, 1))}>‹</button><h3>{month.toLocaleDateString("pt-BR", { month: "long", year: "numeric" })}</h3><button onClick={() => setMonth(new Date(month.getFullYear(), month.getMonth() + 1, 1))}>›</button></header><div className="weekdays">{["Dom", "Seg", "Ter", "Qua", "Qui", "Sex", "Sáb"].map((day) => <b key={day}>{day}</b>)}</div><div className="calendar-days">{Array.from({ length: first }).map((_, index) => <span key={`blank-${index}`} />)}{Array.from({ length: days }).map((_, index) => { const day = index + 1; const key = `${month.getFullYear()}-${String(month.getMonth() + 1).padStart(2, "0")}-${String(day).padStart(2, "0")}`; const item = dated.get(key); return <div key={day} className={item ? "has-event" : ""}><b>{day}</b>{item ? <small>{item.title}</small> : null}</div>; })}</div></div>;
}

function Gallery({ items, admin, onEdit, onDelete, query, onLike, onComment }: { items: PortalItem[]; admin: boolean; onEdit: (item: PortalItem) => void; onDelete: (item: PortalItem) => void; query: string; onLike: (item: PortalItem) => void; onComment: CommentHandler }) {
  if (!items.length) return <div className="empty-state">Nenhuma imagem publicada ainda.</div>;
  return <div className="gallery-grid">{items.map((item) => <GalleryCard key={item.id} item={item} query={query} admin={admin} onEdit={onEdit} onDelete={onDelete} onLike={onLike} onComment={onComment} />)}</div>;
}

function GalleryCard({ item, query, admin, onEdit, onDelete, onLike, onComment }: { item: PortalItem; query: string; admin: boolean; onEdit: (item: PortalItem) => void; onDelete: (item: PortalItem) => void; onLike: (item: PortalItem) => void; onComment: CommentHandler }) {
  const [index, setIndex] = useState(0);
  const files = item.metadata.files || [{ key: item.metadata.key || "", token: item.metadata.token || "", filename: item.metadata.filename || item.title, contentType: item.metadata.contentType || "image/*" }];
  const file = files[index] || files[0];
  return <figure id={`post-${item.id}`}><div className="gallery-media">{file.contentType.startsWith("video/") ? <video src={videoFileUrl(item, query, index)} controls playsInline preload="auto" /> : file.contentType.startsWith("image/") ? <img src={fileUrl(item, query, index)} alt={`${item.title} — ${index + 1}`} /> : <a className="gallery-file" href={fileUrl(item, query, index)} target="_blank">📎<b>{file.filename}</b><span>Abrir arquivo</span></a>}{files.length > 1 ? <><button className="carousel-prev" onClick={() => setIndex((index - 1 + files.length) % files.length)}>‹</button><button className="carousel-next" onClick={() => setIndex((index + 1) % files.length)}>›</button><div className="carousel-count">{index + 1}/{files.length}</div></> : null}</div><figcaption><b>{item.title}</b>{admin ? <span><button onClick={() => onEdit(item)}>Editar</button><button onClick={() => onDelete(item)}>Excluir</button></span> : null}</figcaption><Engagement item={item} onLike={onLike} onComment={onComment} /></figure>;
}

function Materials({ folders, items, admin, onEdit, onDelete, query }: { folders: PortalItem[]; items: PortalItem[]; admin: boolean; onEdit: (item: PortalItem) => void; onDelete: (item: PortalItem) => void; query: string }) {
  const names = [...new Set([...folders.map((item) => item.title), ...items.map((item) => item.metadata.folder || "Materiais gerais")])];
  if (!names.length) return <div className="empty-state">Nenhuma pasta ou material publicado ainda.</div>;
  return <div className="folder-list">{names.map((name) => {
    const folder = folders.find((item) => item.title === name);
    const files = items.filter((item) => (item.metadata.folder || "Materiais gerais") === name);
    return <details key={name} open><summary><span>▰</span><b>{name}</b><small>{files.length} publicação(ões)</small></summary>{folder?.metadata.link ? <a className="folder-link-button" href={folder.metadata.link} target={folder.metadata.link.startsWith("#") ? undefined : "_blank"} rel="noreferrer">Abrir link da pasta ↗</a> : null}{admin && folder ? <div className="folder-actions"><button onClick={() => onEdit(folder)}>Editar pasta e link</button><button onClick={() => onDelete(folder)}>Excluir pasta</button></div> : null}<div>{files.flatMap((item) => (item.metadata.files || [{ filename: item.title, contentType: item.metadata.contentType || "" } as StoredFile]).map((file, index) => <article key={`${item.id}-${index}`}><span>{file.contentType?.startsWith("video/") ? "🎬" : file.contentType?.startsWith("image/") ? "🖼️" : "📎"}</span><a href={fileUrl(item, query, index)} target="_blank">{file.filename || item.title}</a>{admin && index === 0 ? <button onClick={() => onDelete(item)}>Excluir publicação</button> : null}</article>))}</div></details>;
  })}</div>;
}

function Communication({ items, admin, onApprove, onDelete, onMessage, onLike, onComment }: { items: PortalItem[]; admin: boolean; onApprove: (item: PortalItem) => void; onDelete: (item: PortalItem) => void; onMessage: () => void; onLike: (item: PortalItem) => void; onComment: CommentHandler }) {
  return <><div className="communication-intro"><h3>Comunicação e depoimentos</h3><p>Envie orientações, recados e devolutivas. A escola também pode publicar depoimentos, com aprovação da administradora.</p>{!admin ? <button onClick={onMessage}>Escrever recado ou depoimento</button> : null}</div><div className="feed">{items.map((item) => <article id={`post-${item.id}`} className={`post-card${item.metadata.status === "pendente" ? " pending" : ""}${item.kind === "depoimento" ? " testimonial" : ""}`} key={item.id}><div className="post-avatar">{item.kind === "depoimento" ? "♥" : item.kind === "recado" ? "E" : "PC"}</div><div className="post-body"><header><b>{item.kind === "orientacao" ? "POP CHICLÉ PSIEE" : item.title}</b><span>{item.metadata.status === "pendente" ? "Aguardando aprovação" : new Date(item.createdAt).toLocaleDateString("pt-BR")}</span></header><h3>{item.kind === "depoimento" ? "Depoimento" : item.kind === "orientacao" ? item.title : item.metadata.role || "Equipe escolar"}</h3><div className="work-meta">{item.kind === "depoimento" ? <span>♥ Depoimento da escola</span> : null}{item.metadata.priority ? <span>{item.metadata.priority}</span> : null}{item.metadata.audience ? <span>Para: {item.metadata.audience}</span> : null}{item.metadata.date ? <span>📅 {new Date(`${item.metadata.date}T12:00:00`).toLocaleDateString("pt-BR")}</span> : null}</div><p>{item.content}</p><Attachments item={item} />{item.metadata.link ? <a className="work-file" href={item.metadata.link} target="_blank" rel="noreferrer">🔗 Abrir link</a> : null}{admin ? <div className="item-actions">{item.metadata.status === "pendente" ? <button onClick={() => onApprove(item)}>Aprovar</button> : null}<button onClick={() => onDelete(item)}>Excluir</button></div> : null}<Engagement item={item} onLike={onLike} onComment={onComment} /></div></article>)}</div></>;
}

function Attachments({ item }: { item: PortalItem }) {
  return <MediaCarousel item={item} />;
}

function Composer({ section, admin, editing, onClose, onSaveText, onSaveFile, onSaveWork, onSaveCommunication }: { section: Section; admin: boolean; editing: PortalItem | null; onClose: () => void; onSaveText: (event: FormEvent<HTMLFormElement>) => void; onSaveFile: (event: FormEvent<HTMLFormElement>) => void; onSaveWork: (event: FormEvent<HTMLFormElement>) => void; onSaveCommunication: (event: FormEvent<HTMLFormElement>) => void }) {
  const allFiles = "image/*,video/*,.pdf,.doc,.docx,.ppt,.pptx,.xls,.xlsx,.txt,.zip";
  const kind = !admin ? "recado" : section === "trabalhos" ? "trabalho" : section === "comunicacao" ? "orientacao" : section === "materiais" ? "material" : section;
  let form;
  if (section === "trabalhos" && admin) {
    form = <form onSubmit={onSaveWork}><input type="hidden" name="kind" value="trabalho" /><label>Título do trabalho<input name="title" required defaultValue={editing?.title} /></label><label>Descrição completa<textarea name="content" rows={5} required defaultValue={editing?.content} /></label><div className="form-row"><label>Data<input name="date" type="date" defaultValue={editing?.metadata.date} /></label><label>Status<select name="workStatus" defaultValue={editing?.metadata.workStatus || "Em andamento"}><option>Planejado</option><option>Em andamento</option><option>Concluído</option><option>Aguardando devolutiva</option></select></label></div><div className="form-row"><label>Categoria<input name="category" defaultValue={editing?.metadata.category} /></label><label>Responsável<input name="responsible" defaultValue={editing?.metadata.responsible} /></label></div><label>Link relacionado<input name="link" type="url" defaultValue={editing?.metadata.link} /></label>{!editing ? <UploadField label="Até 10 fotos, vídeos ou arquivos — máximo de 1 GB no total" accept={allFiles} /> : null}<button>{editing ? "Salvar alterações" : "Publicar trabalho"}</button></form>;
  } else if (section === "comunicacao") {
    form = <form onSubmit={onSaveCommunication}><label>Tipo de publicação<select name="kind" defaultValue={editing?.kind || (admin ? "orientacao" : "recado")}>{admin ? <><option value="orientacao">Comunicação da assessoria</option><option value="depoimento">Depoimento publicado pela assessoria</option></> : <><option value="recado">Recado para a assessoria</option><option value="depoimento">Depoimento sobre a assessoria</option></>}</select></label><label>{admin ? "Título ou autoria" : "Seu nome"}<input name="title" required defaultValue={editing?.title} /></label><label>Texto da publicação<textarea name="content" rows={6} required defaultValue={editing?.content} /></label><div className="form-row"><label>Data<input name="date" type="date" defaultValue={editing?.metadata.date} /></label><label>Prioridade<select name="priority" defaultValue={editing?.metadata.priority || "Normal"}><option>Normal</option><option>Importante</option><option>Urgente</option></select></label></div><label>Destinatário<input name="audience" defaultValue={editing?.metadata.audience || "Equipe escolar"} /></label><label>Link opcional<input name="link" type="url" defaultValue={editing?.metadata.link} /></label>{!editing ? <UploadField label="Até 10 arquivos, incluindo PNG, JPG, vídeos e documentos — máximo de 1 GB no total" accept={allFiles} /> : null}{!admin ? <label>Função na escola<input name="role" /></label> : null}<button>{editing ? "Salvar alterações" : "Enviar publicação"}</button></form>;
  } else if (section === "materiais" && admin && editing?.kind === "pasta") {
    form = <form onSubmit={onSaveText}><input type="hidden" name="kind" value="pasta" /><label>Nome da pasta<input name="title" required defaultValue={editing.title} /></label><label>Link da pasta<input name="link" defaultValue={editing.metadata.link} /></label><button>Salvar pasta</button></form>;
  } else if (section === "materiais" && admin) {
    form = <><form onSubmit={onSaveText}><input type="hidden" name="kind" value="pasta" /><label>Nome da nova pasta<input name="title" required /></label><label>Link da pasta — opcional<input name="link" /></label><button>Criar pasta</button></form><div className="composer-divider"><span>OU ADICIONE CONTEÚDOS</span></div><form onSubmit={onSaveFile}><input type="hidden" name="kind" value="material" /><label>Título<input name="title" required /></label><label>Pasta<input name="folder" /></label><UploadField label="Até 10 fotos, vídeos ou arquivos — máximo de 1 GB no total" accept={allFiles} required /><button>Publicar materiais</button></form></>;
  } else if (section === "galeria" && editing) {
    form = <form onSubmit={onSaveText}><input type="hidden" name="kind" value="galeria" /><label>Título da postagem<input name="title" required defaultValue={editing.title} /></label><label>Legenda<textarea name="content" rows={4} defaultValue={editing.content} /></label><button>Salvar alterações</button></form>;
  } else if (section === "galeria" && admin) {
    form = <form onSubmit={onSaveFile}><input type="hidden" name="kind" value="galeria" /><label>Título<input name="title" required /></label><label>Legenda<textarea name="content" rows={4} /></label><UploadField label="Carrossel com até 10 fotos ou vídeos — máximo de 1 GB no total" accept="image/*,video/*" required /><button>Publicar na galeria</button></form>;
  } else {
    form = <form onSubmit={onSaveText}><input type="hidden" name="kind" value={kind} /><label>Título<input name="title" required defaultValue={editing?.title} /></label>{section === "agenda" ? <label>Data<input name="date" type="date" defaultValue={editing?.metadata.date} /></label> : null}<label>Descrição<textarea name="content" rows={6} required defaultValue={editing?.content} /></label><button>{editing ? "Salvar alterações" : "Publicar"}</button></form>;
  }
  return <div className="modal-backdrop" onMouseDown={(event) => { if (event.target === event.currentTarget) onClose(); }}><section className="composer"><header><div><span>{editing ? "EDITAR PUBLICAÇÃO" : "NOVA PUBLICAÇÃO"}</span><h2>{sections.find((item) => item.id === section)?.label}</h2></div><button onClick={onClose}>×</button></header>{form}</section></div>;
}

function UploadField({ label, accept, required = false }: { label: string; accept: string; required?: boolean }) {
  const [selectedNames, setSelectedNames] = useState<string[]>([]);
  return <label className="upload-field">{label}<input name="files" type="file" multiple required={required} accept={accept} onChange={(event) => setSelectedNames(Array.from(event.currentTarget.files || []).map((file) => file.name))} />{selectedNames.length ? <div className="selected-files"><b>✓ {selectedNames.length} arquivo{selectedNames.length === 1 ? "" : "s"} selecionado{selectedNames.length === 1 ? "" : "s"}</b>{selectedNames.map((name) => <span key={name}>{name}</span>)}</div> : <small>Selecione até 10 fotos, vídeos ou arquivos de uma vez.</small>}</label>;
}

```

## app/globals.css

```css
:root {
  --blue: #5b76e8;
  --blue-dark: #344eb5;
  --pink: #ff4f9b;
  --yellow: #ffd84a;
  --lime: #c8f267;
  --ink: #22304f;
  --muted: #6d7891;
  --line: #dbe1ee;
  --paper: #ffffff;
  --bg: #eef1f8;
}

* { box-sizing: border-box; }
html, body { min-height: 100%; margin: 0; }
body { color: var(--ink); background: var(--bg); font-family: Arial, Helvetica, sans-serif; }
button, input, textarea, select { font: inherit; }
button, a { -webkit-tap-highlight-color: transparent; }
button { cursor: pointer; }

.auth-page {
  min-height: 100dvh;
  display: grid;
  place-items: center;
  padding: 24px;
  background:
    radial-gradient(circle at 12% 12%, rgba(255,216,74,.55), transparent 28rem),
    radial-gradient(circle at 88% 8%, rgba(255,79,155,.28), transparent 30rem),
    linear-gradient(145deg, #49b9f4, #6273ea 58%, #8e4fea);
}

.auth-card {
  width: min(100%, 580px);
  padding: clamp(28px, 5vw, 54px);
  border: 3px solid var(--ink);
  border-radius: 30px;
  background: #fffdf8;
  box-shadow: 14px 16px 0 var(--ink);
}

.auth-logo { display: flex; align-items: baseline; gap: 8px; font-weight: 900; }
.auth-logo b { color: var(--pink); font-size: 1.8rem; }
.auth-logo strong { font-size: 1.4rem; }
.auth-logo span { padding: 3px 7px; border-radius: 999px; color: white; background: var(--blue); font-size: .7rem; }
.eyebrow { margin: 34px 0 8px; color: var(--blue-dark); font-size: .7rem; font-weight: 900; letter-spacing: .15em; }
.auth-card h1 { margin: 0; font-family: Georgia, serif; font-size: clamp(2.7rem, 8vw, 5.2rem); line-height: .88; letter-spacing: -.06em; }
.auth-card > p:not(.eyebrow) { color: var(--muted); line-height: 1.6; }
.auth-card form, .composer form { display: grid; gap: 14px; margin-top: 28px; }
.auth-card label, .composer label, .create-school label { display: grid; gap: 7px; font-size: .76rem; font-weight: 900; }
.auth-card input, .composer input, .composer textarea, .composer select, .create-school input {
  min-width: 0; padding: 14px; border: 2px solid var(--line); border-radius: 12px; background: white; outline: none;
}
.auth-card input:focus, .composer input:focus, .composer textarea:focus, .composer select:focus, .create-school input:focus { border-color: var(--blue); box-shadow: 0 0 0 4px rgba(91,118,232,.15); }
.auth-card form button, .composer form button, .create-school button {
  padding: 14px 18px; border: 2px solid var(--ink); border-radius: 12px; color: var(--ink); background: var(--yellow); font-weight: 900; box-shadow: 4px 4px 0 var(--ink);
}
.form-error { color: #b41450 !important; font-weight: 800; }
.switch-login { width: 100%; margin-top: 14px; padding: 10px; border: 0; color: var(--blue-dark); background: transparent; font-weight: 900; }

.admin-home { min-height: 100dvh; padding-bottom: 60px; background: #f5f6fb; }
.admin-home > header, .orkut-topbar {
  min-height: 72px; display: flex; align-items: center; justify-content: space-between; gap: 20px; padding: 12px clamp(18px, 4vw, 56px); color: white; background: linear-gradient(90deg, #34afd8, var(--blue) 58%, #7c4ce1);
}
.admin-home > header > div:last-child { display: grid; text-align: right; }
.admin-header-actions { display: flex !important; align-items: center; gap: 12px; }
.admin-header-actions > div { display: grid; }
.admin-header-actions > button { padding: 9px 16px; border: 1px solid rgba(255,255,255,.6); border-radius: 9px; color: white; background: transparent; font-weight: 900; }
.admin-home > header span { font-size: .72rem; opacity: .85; }
.brand-word { font-size: 1.25rem; font-weight: 900; letter-spacing: -.04em; white-space: nowrap; }
.brand-word b { color: var(--yellow); }
.brand-word span { padding: 3px 6px; border-radius: 999px; color: white; background: var(--pink); font-size: .62rem; letter-spacing: .08em; }
.admin-intro, .create-school, .school-grid { width: min(1180px, calc(100% - 36px)); margin-inline: auto; }
.admin-intro { padding: 46px 0 24px; }
.admin-intro .eyebrow { margin: 0; }
.admin-intro h1 { margin: 5px 0; font-family: Georgia, serif; font-size: clamp(2.6rem, 6vw, 5.2rem); letter-spacing: -.06em; }
.admin-intro p:last-child { color: var(--muted); }
.create-school { display: grid; grid-template-columns: minmax(0, 1.2fr) minmax(260px, .8fr); gap: 18px; padding: 22px; border-radius: 20px; color: white; background: var(--ink); }
.create-school form { display: grid; grid-template-columns: 1fr auto; align-items: end; gap: 12px; }
.created-code { display: grid; grid-template-columns: 1fr auto; align-items: center; gap: 6px 12px; padding: 14px; border-radius: 12px; color: var(--ink); background: white; }
.created-code span { grid-column: 1 / -1; font-size: .64rem; font-weight: 900; letter-spacing: .12em; }
.created-code strong { font-size: 1.45rem; letter-spacing: .08em; }
.created-code button { padding: 8px 10px; box-shadow: none; }
.school-grid { display: grid; gap: 14px; margin-top: 20px; }
.school-grid article {
  display: grid; grid-template-columns: 70px minmax(0, 1fr) auto auto; align-items: center; gap: 16px; padding: 20px; border: 1px solid var(--line); border-radius: 16px; background: white; box-shadow: 0 7px 20px rgba(37,49,86,.06);
}
.folder-icon { width: 60px; height: 48px; display: grid; place-items: center; border: 2px solid var(--ink); border-radius: 10px; color: var(--ink); background: var(--pink); font-size: 1.8rem; }
.school-grid small { color: var(--blue-dark); font-weight: 900; }
.school-grid h2 { margin: 3px 0; }
.school-grid p { margin: 0; color: var(--muted); }
.school-grid button { padding: 11px 14px; border: 0; border-radius: 10px; color: white; background: var(--blue); font-weight: 900; }
.school-grid .secondary { color: var(--blue-dark); background: #edf0ff; }
.admin-orkut-layout { width: min(1420px, calc(100% - 28px)); margin: 18px auto 50px; display: grid; grid-template-columns: 230px minmax(0, 1fr) 250px; gap: 16px; align-items: start; }
.admin-profile, .admin-welcome, .admin-right { border: 1px solid var(--line); border-radius: 16px; background: white; box-shadow: 0 5px 16px rgba(39,53,92,.06); }
.admin-profile { padding: 18px; text-align: center; }
.admin-profile h2 { margin: 12px 0 4px; }
.admin-profile p { margin: 0; color: var(--muted); font-size: .78rem; line-height: 1.45; }
.admin-profile > button, .admin-right button { width: 100%; margin-top: 14px; padding: 10px; border: 0; border-radius: 9px; color: white; background: var(--blue); font-weight: 900; }
.admin-profile nav { display: grid; gap: 5px; margin-top: 18px; padding-top: 12px; border-top: 1px solid var(--line); text-align: left; }
.admin-profile nav > * { padding: 9px; border-radius: 8px; color: #4c5873; }
.admin-profile nav b { color: var(--blue-dark); background: #edf0ff; }
.admin-center { min-width: 0; display: grid; gap: 14px; }
.admin-welcome { overflow: hidden; padding: 25px; }
.admin-welcome small { color: var(--blue-dark); font-weight: 900; }
.admin-welcome h1 { margin: 10px 0 6px; font-family: Georgia, serif; font-size: clamp(2.2rem, 5vw, 4rem); letter-spacing: -.05em; }
.admin-welcome > p { color: var(--muted); }
.admin-welcome > div { display: flex; flex-wrap: wrap; gap: 24px; margin-top: 20px; padding-top: 18px; border-top: 1px solid var(--line); }
.admin-welcome > div span { display: grid; color: var(--muted); font-size: .7rem; }
.admin-welcome > div b { color: var(--blue-dark); font-size: 1.3rem; }
.admin-orkut-layout .create-school, .admin-orkut-layout .school-grid { width: 100%; margin: 0; }
.admin-right { padding: 18px; }
.admin-right h3 { margin-top: 0; color: var(--blue-dark); }
.admin-right p { color: var(--muted); line-height: 1.5; }
.admin-right p b { color: var(--pink); font-size: 1.8rem; }
.delete-photo { width: 100%; margin-top: 6px; padding: 7px; border: 0; color: #b3154c; background: transparent; font-size: .68rem; font-weight: 900; }
.edit-school-profile { width: 100%; margin-top: 6px; padding: 8px; border: 0; border-radius: 8px; color: var(--blue-dark); background: #edf0ff; font-size: .7rem; font-weight: 900; }
.color-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; }
.color-grid input[type="color"] { width: 100%; height: 48px; padding: 4px; }

.orkut-shell { min-height: 100dvh; overflow-x: hidden; background: var(--bg); }
.orkut-topbar { position: sticky; z-index: 100; top: 0; min-height: 64px; padding: 8px clamp(12px, 3vw, 40px); box-shadow: 0 4px 18px rgba(38,48,88,.18); }
.top-school { flex: 1; display: grid; text-align: right; }
.top-school span { font-size: .7rem; opacity: .82; }
.top-actions { display: flex; gap: 8px; }
.top-actions button, .mobile-menu { padding: 9px 12px; border: 1px solid rgba(255,255,255,.5); border-radius: 9px; color: white; background: rgba(255,255,255,.12); font-weight: 800; }
.mobile-menu { display: none; }
.mobile-logout { display: none !important; }
.orkut-layout { width: min(1420px, calc(100% - 24px)); margin: 18px auto 50px; display: grid; grid-template-columns: 230px minmax(0, 1fr) 260px; gap: 16px; align-items: start; }
.left-column, .right-column { position: sticky; top: 82px; display: grid; gap: 14px; }
.school-profile, .advisor-card, .summary-card, .method-card, .section-head, .orkut-home-card, .post-card, .calendar-card, .communication-intro, .agenda-list, .empty-state {
  border: 1px solid var(--line); border-radius: 16px; background: white; box-shadow: 0 5px 16px rgba(39,53,92,.055);
}
.school-profile { padding: 18px; text-align: center; }
.school-avatar { aspect-ratio: 1; display: grid; place-items: center; border-radius: 12px; background: linear-gradient(145deg, var(--yellow), #ff9d55); font-size: 4rem; }
.school-avatar { overflow: hidden; }
.school-avatar img { width: 100%; height: 100%; display: block; object-fit: cover; }
.school-profile h1 { margin: 13px 0 5px; font-size: 1.1rem; }
.school-profile p { margin: 0; color: var(--muted); font-size: .76rem; line-height: 1.4; }
.change-photo { display: block; margin-top: 11px; padding: 8px 10px; border-radius: 8px; color: var(--blue-dark); background: #edf0ff; font-size: .7rem; font-weight: 900; cursor: pointer; }
.change-photo input { position: absolute; width: 1px; height: 1px; opacity: 0; pointer-events: none; }
.upload-field > small { color: var(--muted); font-weight: 600; line-height: 1.4; }
.selected-files { display: grid; gap: 5px; padding: 10px 12px; border-radius: 10px; color: var(--blue-dark); background: #edf0ff; }
.selected-files span { overflow: hidden; font-weight: 600; text-overflow: ellipsis; white-space: nowrap; }
.orkut-nav { display: grid; padding: 8px; border: 1px solid var(--line); border-radius: 14px; background: white; }
.orkut-nav button { display: flex; align-items: center; gap: 10px; min-height: 42px; padding: 8px 10px; border: 0; border-radius: 9px; color: #4c5873; background: transparent; text-align: left; font-weight: 700; }
.orkut-nav button span { width: 25px; color: var(--blue); text-align: center; font-size: 1.1rem; }
.orkut-nav button:hover, .orkut-nav button.active { color: var(--blue-dark); background: #edf0ff; }
.access-note { padding: 14px; border-radius: 14px; color: #624300; background: #fff2b5; font-size: .76rem; }
.access-note p { margin: 6px 0 0; line-height: 1.45; }
.center-column { min-width: 0; display: grid; gap: 14px; }
.section-head { min-height: 88px; display: flex; align-items: center; justify-content: space-between; gap: 16px; padding: 18px 22px; }
.section-head span { color: var(--muted); font-size: .72rem; }
.section-head h2 { margin: 3px 0 0; font-family: Georgia, serif; font-size: 2rem; }
.section-head button, .communication-intro button { padding: 10px 14px; border: 0; border-radius: 10px; color: white; background: var(--blue); font-weight: 900; }
.orkut-home-card { overflow: hidden; }
.orkut-home-card > header { padding: 10px 16px; color: var(--blue-dark); background: #e4eaff; font-size: .8rem; font-weight: 900; }
.orkut-home-card > div { padding: 24px; }
.orkut-home-card .orkut-hello { margin: 0 0 7px; color: var(--ink); font-family: Georgia, serif; font-size: clamp(2rem, 5vw, 3.5rem); font-weight: 700; letter-spacing: -.04em; }
.orkut-home-card p { margin: 0; color: var(--muted); }
.orkut-home-card section { display: flex; flex-wrap: wrap; gap: 24px; margin-top: 22px; padding-top: 18px; border-top: 1px solid var(--line); }
.orkut-home-card section span { display: grid; color: var(--muted); font-size: .72rem; }
.orkut-home-card section b { color: var(--blue-dark); font-size: 1.4rem; }
.shortcut-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
.shortcut-grid button { display: grid; grid-template-columns: 48px 1fr; gap: 2px 10px; align-items: center; min-height: 86px; padding: 13px; border: 1px solid var(--line); border-radius: 13px; color: var(--ink); background: white; text-align: left; }
.shortcut-grid button > span { grid-row: 1 / 3; width: 48px; height: 48px; display: grid; place-items: center; border-radius: 12px; color: var(--blue-dark); background: #e8edff; font-size: 1.4rem; }
.shortcut-grid small { color: var(--muted); }
.feed-title h3 { margin: 10px 2px 0; }
.feed { display: grid; gap: 12px; }
.post-card { display: grid; grid-template-columns: 48px 1fr; gap: 12px; padding: 16px; }
.post-card.pending { border-color: #f2b72c; background: #fffbeb; }
.post-card.testimonial { border-left: 5px solid var(--pink); }
.post-avatar, .advisor-avatar { display: grid; place-items: center; border-radius: 50%; color: white; background: linear-gradient(145deg, var(--pink), #8c4be0); font-weight: 900; }
.post-avatar { width: 46px; height: 46px; }
.post-body { min-width: 0; }
.post-body header { display: flex; justify-content: space-between; gap: 12px; }
.post-body header b { color: var(--blue-dark); font-size: .78rem; }
.post-body header span { color: var(--muted); font-size: .68rem; }
.post-body h3 { margin: 9px 0 5px; }
.post-body p { margin: 0; color: #4d5870; line-height: 1.55; white-space: pre-wrap; overflow-wrap: anywhere; }
.work-details { display: grid; gap: 11px; }
.work-meta { display: flex; flex-wrap: wrap; gap: 6px; }
.work-meta span { padding: 6px 9px; border-radius: 999px; color: var(--blue-dark); background: #edf0ff; font-size: .7rem; font-weight: 800; }
.work-image { width: min(100%, 680px); max-height: 560px; object-fit: contain; border-radius: 10px; background: #f4f6fb; }
.portal-media-carousel { position: relative; width: 100%; aspect-ratio: 4 / 3; overflow: hidden; border-radius: 10px; background: #f4f5f9; }
.portal-media-carousel > img, .portal-media-carousel > video { width: 100%; height: 100%; display: block; border: 0; object-fit: cover; }
.portal-media-carousel > canvas { display: block; width: auto; height: auto; max-width: 100%; max-height: 100%; margin: auto; background: white; }
.pdf-fallback { width: 100%; height: 100%; display: grid; place-items: center; align-content: center; gap: 6px; color: var(--blue-dark); background: white; text-align: center; font-weight: 900; }
.pdf-fallback small { max-width: 80%; color: var(--muted); overflow-wrap: anywhere; }
.portal-media-carousel .gallery-file { position: absolute; inset: 0; }
.media-open { position: absolute; z-index: 3; left: 10px; bottom: 10px; padding: 7px 10px; border-radius: 999px; color: white; background: rgba(20,29,55,.8); text-decoration: none; font-size: .72rem; font-weight: 900; }
.work-file { width: fit-content; padding: 9px 11px; border-radius: 9px; color: var(--blue-dark); background: #edf0ff; text-decoration: none; font-size: .78rem; font-weight: 900; }
.item-actions { display: flex; gap: 7px; margin-top: 12px; }
.item-actions button { padding: 6px 9px; border: 0; border-radius: 7px; color: var(--blue-dark); background: #edf0ff; font-size: .72rem; font-weight: 800; }
.item-actions button:last-child { color: #b3154c; background: #fff0f5; }
.engagement { margin-top: 14px; padding-top: 11px; border-top: 1px solid #edf0f6; }
.social-actions { display: flex; flex-wrap: wrap; gap: 6px; }
.social-actions button, .share-button { padding: 7px 10px; border: 0; border-radius: 999px; color: var(--blue-dark); background: #edf0ff; font-size: .74rem; font-weight: 900; }
.social-actions button.liked { color: #b3154c; background: #fff0f5; }
.social-actions button span { margin-left: 4px; }
.comment-list { display: grid; gap: 6px; margin-top: 10px; }
.comment-list p { display: grid; gap: 2px; margin: 0; padding: 9px 11px; border-radius: 9px; background: #f5f7fb; font-size: .78rem; }
.comment-list p b { color: var(--blue-dark); }
.engagement form { display: grid; grid-template-columns: minmax(105px, .42fr) minmax(0, 1fr) auto; gap: 7px; margin-top: 10px; }
.engagement form input { min-width: 0; padding: 9px 11px; border: 1px solid var(--line); border-radius: 9px; outline: none; }
.engagement form input:focus { border-color: var(--blue); }
.engagement form button { padding: 9px 11px; border: 0; border-radius: 9px; color: white; background: var(--blue); font-size: .72rem; font-weight: 900; }
.gallery-grid .engagement { margin: 0; padding: 0 12px 12px; border: 0; }
.agenda-list { overflow: hidden; }
.agenda-list article { display: grid; grid-template-columns: 88px 1fr; gap: 18px; padding: 19px; border-bottom: 1px solid var(--line); }
.agenda-list article:last-child { border-bottom: 0; }
.agenda-list time { align-self: start; padding: 10px; border-radius: 10px; color: var(--blue-dark); background: #edf0ff; text-align: center; font-size: .8rem; font-weight: 900; text-transform: uppercase; }
.agenda-list h3 { margin: 0 0 5px; }
.agenda-list p { margin: 0; color: var(--muted); line-height: 1.5; }
.agenda-list .share-button { margin-top: 10px; }
.calendar-card { overflow: hidden; padding: 18px; }
.calendar-card > header { display: flex; align-items: center; justify-content: space-between; }
.calendar-card > header button { width: 36px; height: 36px; border: 0; border-radius: 50%; color: var(--blue-dark); background: #edf0ff; font-size: 1.3rem; }
.calendar-card h3 { text-transform: capitalize; }
.weekdays, .calendar-days { display: grid; grid-template-columns: repeat(7, minmax(0, 1fr)); }
.weekdays b { padding: 10px 3px; color: var(--muted); text-align: center; font-size: .68rem; }
.calendar-days > div, .calendar-days > span { min-height: 94px; padding: 8px; border: 1px solid #edf0f6; }
.calendar-days > div b { font-size: .76rem; }
.calendar-days > div small { display: block; margin-top: 9px; padding: 5px; border-radius: 6px; color: var(--blue-dark); background: #edf0ff; font-size: .64rem; overflow: hidden; text-overflow: ellipsis; }
.calendar-days > div.has-event { background: #fbfcff; }
.gallery-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; }
.gallery-grid figure { margin: 0; overflow: hidden; border: 1px solid var(--line); border-radius: 14px; background: white; }
.gallery-media { position: relative; width: 100%; aspect-ratio: 4 / 3; overflow: hidden; background: #f4f5f9; }
.gallery-grid img, .gallery-grid video { width: 100%; height: 100%; display: block; object-fit: cover; }
.gallery-grid video::-webkit-media-controls-fullscreen-button,
.work-image::-webkit-media-controls-fullscreen-button,
.portal-media-carousel video::-webkit-media-controls-fullscreen-button,
.communication-files video::-webkit-media-controls-fullscreen-button { display: none; }
.gallery-file { width: 100%; height: 100%; display: grid; place-items: center; align-content: center; gap: 8px; color: var(--blue-dark); text-decoration: none; }
.gallery-file:first-letter { font-size: 3rem; }
.gallery-file span { color: var(--muted); font-size: .75rem; }
.carousel-prev, .carousel-next { position: absolute; z-index: 2; top: 50%; width: 38px; height: 38px; border: 0; border-radius: 50%; color: white; background: rgba(20,29,55,.72); font-size: 1.6rem; transform: translateY(-50%); }
.carousel-prev { left: 10px; }
.carousel-next { right: 10px; }
.carousel-count { position: absolute; z-index: 2; top: 10px; right: 10px; padding: 5px 8px; border-radius: 999px; color: white; background: rgba(20,29,55,.72); font-size: .68rem; font-weight: 900; }
.gallery-grid figcaption { display: flex; justify-content: space-between; gap: 10px; padding: 12px; }
.gallery-grid figcaption button, .folder-list article button { border: 0; color: #b3154c; background: transparent; font-size: .72rem; font-weight: 800; }
.folder-list { display: grid; gap: 12px; }
.folder-list details { border: 1px solid var(--line); border-radius: 14px; background: white; }
.folder-list summary { display: grid; grid-template-columns: 40px 1fr auto; align-items: center; gap: 9px; padding: 15px; cursor: pointer; list-style: none; }
.folder-list summary > span { width: 38px; height: 32px; display: grid; place-items: center; border-radius: 7px; background: var(--yellow); }
.folder-list summary small { color: var(--muted); }
.folder-list details > div { display: grid; border-top: 1px solid var(--line); }
.folder-link-button { display: block; margin: 0 15px 12px; padding: 10px 12px; border-radius: 9px; color: white !important; background: var(--blue); text-align: center; text-decoration: none; font-size: .78rem; font-weight: 900; }
.folder-actions { display: flex !important; gap: 7px; padding: 0 15px 12px; border: 0 !important; }
.folder-actions button { padding: 7px 9px; border: 0; border-radius: 7px; color: var(--blue-dark); background: #edf0ff; font-size: .7rem; font-weight: 800; }
.folder-actions button:last-child { color: #b3154c; background: #fff0f5; }
.folder-list article { display: grid; grid-template-columns: 28px 1fr auto; align-items: center; gap: 8px; padding: 12px 15px; border-bottom: 1px solid #eef1f6; }
.folder-list article:last-child { border: 0; }
.folder-list a { color: var(--blue-dark); font-weight: 700; text-decoration: none; }
.communication-intro { padding: 22px; }
.communication-intro h3 { margin: 0; }
.communication-intro p { color: var(--muted); line-height: 1.5; }
.communication-files { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 8px; margin-top: 10px; }
.communication-files img, .communication-files video { width: 100%; max-height: 340px; object-fit: cover; border-radius: 9px; background: #f4f5f9; }
.communication-files a { padding: 11px; border-radius: 9px; color: var(--blue-dark); background: #edf0ff; text-decoration: none; font-weight: 800; }
.work-attachment { display: grid; gap: 7px; }
.advisor-card, .summary-card, .method-card { padding: 17px; text-align: center; }
.advisor-avatar { width: 76px; height: 76px; margin: auto; font-size: 1.4rem; }
.advisor-card h2 { margin: 12px 0 4px; font-size: 1rem; }
.advisor-card p { margin: 0; color: var(--muted); font-size: .74rem; }
.online { display: block; margin-top: 12px; color: #3d8d39; font-size: .68rem; font-weight: 800; }
.summary-card { display: grid; grid-template-columns: repeat(3, 1fr); gap: 7px; }
.summary-card h3 { grid-column: 1 / -1; margin: 0 0 7px; text-align: left; font-size: .86rem; }
.summary-card div { display: grid; gap: 3px; padding: 9px 2px; border-radius: 9px; background: #f3f5fb; }
.summary-card b { color: var(--blue-dark); font-size: 1.1rem; }
.summary-card span { color: var(--muted); font-size: .56rem; }
.method-card { color: #553d00; background: var(--yellow); }
.method-card p { margin: 4px 0 0; font-size: .75rem; }
.empty-state { padding: 38px 20px; color: var(--muted); text-align: center; }
.modal-backdrop { position: fixed; z-index: 500; inset: 0; display: grid; place-items: center; padding: 18px; background: rgba(21,30,58,.62); }
.composer { width: min(100%, 560px); max-height: calc(100dvh - 36px); overflow: auto; padding: 22px; border-radius: 20px; background: white; box-shadow: 0 25px 70px rgba(0,0,0,.3); }
.composer > header { display: flex; justify-content: space-between; gap: 20px; }
.composer header span { color: var(--blue-dark); font-size: .65rem; font-weight: 900; letter-spacing: .12em; }
.composer h2 { margin: 4px 0; font-family: Georgia, serif; font-size: 2rem; }
.composer header button { width: 38px; height: 38px; border: 0; border-radius: 50%; background: #eef1f7; font-size: 1.4rem; }
.composer textarea { resize: vertical; }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.composer-divider { position: relative; margin: 22px 0 2px; border-top: 1px solid var(--line); text-align: center; }
.composer-divider span { position: relative; top: -8px; padding: 0 10px; color: var(--muted); background: white; font-size: .62rem; font-weight: 900; letter-spacing: .1em; }
.field-help { margin: -2px 0 0; color: var(--muted); font-size: .72rem; line-height: 1.45; }
.portal-toast { position: fixed; z-index: 600; left: 50%; bottom: 24px; padding: 12px 18px; border-radius: 10px; color: white; background: var(--ink); font-weight: 800; transform: translateX(-50%); box-shadow: 0 8px 30px rgba(0,0,0,.2); }
.shared-page { min-height: 100dvh; display: grid; place-items: center; padding: 24px; background: var(--bg); }
.shared-page article { width: min(100%, 760px); overflow: hidden; border: 1px solid var(--line); border-radius: 18px; background: white; box-shadow: 0 15px 50px rgba(39,53,92,.14); }
.shared-page article > header { display: flex; justify-content: space-between; gap: 16px; padding: 15px 20px; color: white; background: linear-gradient(90deg, #34afd8, var(--blue), #7c4ce1); }
.shared-page article > img, .shared-page article > video { width: 100%; max-height: 620px; display: block; object-fit: contain; background: #f4f5f9; }
.shared-page article > div { padding: clamp(22px, 5vw, 42px); }
.shared-page small { color: var(--blue-dark); font-weight: 900; letter-spacing: .1em; }
.shared-page h1 { margin: 8px 0; font-family: Georgia, serif; font-size: clamp(2rem, 6vw, 4rem); }
.shared-page p { color: var(--muted); line-height: 1.65; white-space: pre-wrap; }
.shared-page a { display: inline-block; margin-top: 15px; padding: 11px 15px; border-radius: 9px; color: white; background: var(--blue); text-decoration: none; font-weight: 900; }

@media (max-width: 1080px) {
  .orkut-layout { grid-template-columns: 210px minmax(0, 1fr); }
  .right-column { display: none; }
  .admin-orkut-layout { grid-template-columns: 220px minmax(0, 1fr); }
  .admin-right { display: none; }
}

@media (max-width: 760px) {
  .orkut-topbar { min-height: 58px; }
  .top-logout { display: none; }
  .mobile-logout { display: flex !important; margin-top: 6px; border-top: 1px solid var(--line) !important; color: #b3154c !important; }
  .mobile-menu { display: block; }
  .brand-word { font-size: 1rem; }
  .top-school { display: none; }
  .orkut-layout { width: min(100% - 16px, 760px); grid-template-columns: 1fr; margin-top: 8px; }
  .left-column { position: fixed; z-index: 200; top: 58px; left: 0; bottom: 0; width: min(86vw, 310px); padding: 10px; overflow: auto; background: var(--bg); transform: translateX(-105%); transition: transform .2s ease; box-shadow: 10px 0 30px rgba(25,35,70,.2); }
  .left-column.open { transform: translateX(0); }
  .school-profile { display: grid; grid-template-columns: 70px 1fr; gap: 10px; align-items: center; text-align: left; }
  .school-avatar { grid-row: 1 / 3; font-size: 2.4rem; }
  .school-profile h1 { margin: 0; }
  .shortcut-grid { grid-template-columns: repeat(2, 1fr); }
  .create-school { grid-template-columns: 1fr; }
  .school-grid article { grid-template-columns: 55px 1fr; }
  .school-grid article button { grid-column: auto; }
  .folder-icon { width: 50px; height: 42px; }
  .admin-orkut-layout { grid-template-columns: 1fr; }
  .admin-profile { display: grid; grid-template-columns: 70px 1fr; gap: 8px 12px; text-align: left; }
  .admin-profile .advisor-avatar { grid-row: 1 / 4; width: 70px; height: 70px; }
  .admin-profile h2, .admin-profile p { margin: 0; }
  .admin-profile > button, .admin-profile nav { grid-column: 1 / -1; }
}

@media (max-width: 520px) {
  .auth-card { padding: 25px; border-radius: 22px; box-shadow: 8px 9px 0 var(--ink); }
  .auth-card h1 { font-size: 3rem; }
  .section-head { min-height: 74px; padding: 14px; }
  .section-head h2 { font-size: 1.6rem; }
  .section-head button { padding: 9px; font-size: .78rem; }
  .orkut-home-card > div { padding: 18px; }
  .engagement form { grid-template-columns: 1fr; }
  .form-row { grid-template-columns: 1fr; }
  .communication-files { grid-template-columns: 1fr; }
  .shortcut-grid { grid-template-columns: 1fr; }
  .post-card { grid-template-columns: 38px 1fr; padding: 13px; }
  .post-avatar { width: 38px; height: 38px; font-size: .7rem; }
  .post-body header { display: grid; gap: 3px; }
  .agenda-list article { grid-template-columns: 65px 1fr; gap: 12px; padding: 14px; }
  .calendar-card { padding: 10px; overflow-x: auto; }
  .weekdays, .calendar-days { min-width: 600px; }
  .calendar-days > div, .calendar-days > span { min-height: 78px; }
  .gallery-grid { grid-template-columns: 1fr; }
  .create-school form { grid-template-columns: 1fr; }
  .school-grid article { grid-template-columns: 48px 1fr; padding: 15px; }
  .school-grid article button { grid-column: 1 / -1; }
  .created-code { grid-template-columns: 1fr; }
  .color-grid { grid-template-columns: repeat(2, 1fr); }
}

```

## app/api/access/route.ts

```ts
import { desc, eq } from "drizzle-orm";
import { getDb } from "../../../db";
import { portalItems, schoolSessions, schools } from "../../../db/schema";
import {
  getSchoolSession,
  ADMIN_COOKIE,
  isAdminRequest,
  newSchoolCode,
  readCookie,
  SESSION_COOKIE,
  sha256,
} from "../school-auth";

export const dynamic = "force-dynamic";

function cookie(token: string, maxAge: number) {
  return `${SESSION_COOKIE}=${encodeURIComponent(token)}; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=${maxAge}`;
}

function adminCookie(token: string, maxAge: number) {
  return `${ADMIN_COOKIE}=${encodeURIComponent(token)}; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=${maxAge}`;
}

function withCookies(...values: string[]) {
  const headers = new Headers();
  for (const value of values) headers.append("set-cookie", value);
  return headers;
}

export async function GET(request: Request) {
  if (new URL(request.url).searchParams.get("entry") === "1") {
    return Response.json({ role: "guest" });
  }
  const db = await getDb();
  if (await isAdminRequest(request)) {
    const list = await db.select({ id: schools.id, name: schools.name, code: schools.accessCode, createdAt: schools.createdAt }).from(schools).orderBy(desc(schools.createdAt));
    return Response.json({ role: "admin", schools: list });
  }
  const school = await getSchoolSession(request);
  if (school) return Response.json({ role: "school", school: { id: school.id, name: school.name } });
  return Response.json({ role: "guest" });
}

export async function POST(request: Request) {
  const body = (await request.json()) as { action?: string; name?: string; subtitle?: string; code?: string; schoolId?: string; username?: string; password?: string };
  const db = await getDb();

  if (body.action === "admin-login") {
    if (body.username !== process.env.ADMIN_USERNAME || body.password !== process.env.ADMIN_PASSWORD) {
      return Response.json({ error: "Usuário ou senha incorretos." }, { status: 401 });
    }
    return Response.json(
      { ok: true },
      { headers: withCookies(adminCookie(process.env.ADMIN_SESSION_TOKEN || "", 30 * 24 * 60 * 60), cookie("", 0)) },
    );
  }

  if (body.action === "create-school") {
    if (!(await isAdminRequest(request))) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
    const name = body.name?.trim();
    if (!name) return Response.json({ error: "Informe o nome da escola." }, { status: 400 });
    const code = newSchoolCode();
    const school = { id: crypto.randomUUID(), name, codeHash: await sha256(code), accessCode: code };
    await db.insert(schools).values(school);
    await db.insert(portalItems).values([
      {
        schoolId: school.id,
        kind: "trabalho",
        title: "Acolhimento e retomada",
        content: "Apresentação da nova fase da assessoria, organização dos canais de comunicação e definição das prioridades iniciais.",
        metadata: JSON.stringify({ status: "publicado" }),
        sortOrder: 3,
      },
      {
        schoolId: school.id,
        kind: "trabalho",
        title: "Leitura institucional",
        content: "Levantamento das demandas, barreiras, potencialidades, rotinas e necessidades da equipe escolar.",
        metadata: JSON.stringify({ status: "publicado" }),
        sortOrder: 2,
      },
      {
        schoolId: school.id,
        kind: "orientacao",
        title: "Bem-vindos ao portal",
        content: "Este espaço será utilizado para organizar a agenda, registrar as ações, disponibilizar materiais e manter a comunicação com a equipe escolar.",
        metadata: JSON.stringify({ status: "publicado" }),
        sortOrder: 1,
      },
    ]);
    return Response.json({ school: { id: school.id, name }, code }, { status: 201 });
  }

  if (body.action === "reset-code") {
    if (!(await isAdminRequest(request))) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
    const schoolId = body.schoolId?.trim();
    if (!schoolId) return Response.json({ error: "Escola não informada." }, { status: 400 });
    const code = newSchoolCode();
    await db.update(schools).set({ codeHash: await sha256(code), accessCode: code, updatedAt: new Date().toISOString() }).where(eq(schools.id, schoolId));
    return Response.json({ code });
  }

  if (body.action === "remember-code") {
    if (!(await isAdminRequest(request))) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
    const schoolId = body.schoolId?.trim();
    const code = body.code?.trim().toUpperCase();
    if (!schoolId || !code) return Response.json({ error: "Código não informado." }, { status: 400 });
    const [school] = await db.select({ codeHash: schools.codeHash }).from(schools).where(eq(schools.id, schoolId)).limit(1);
    if (!school || school.codeHash !== await sha256(code)) return Response.json({ error: "Este não é o código atual da escola." }, { status: 400 });
    await db.update(schools).set({ accessCode: code, updatedAt: new Date().toISOString() }).where(eq(schools.id, schoolId));
    return Response.json({ code, ok: true });
  }

  if (body.action === "update-school") {
    if (!(await isAdminRequest(request))) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
    const schoolId = body.schoolId?.trim();
    const name = body.name?.trim();
    const subtitle = String((body as { subtitle?: string }).subtitle || "").trim();
    if (!schoolId || !name) return Response.json({ error: "Informe o nome da escola." }, { status: 400 });
    await db.update(schools).set({ name, updatedAt: new Date().toISOString() }).where(eq(schools.id, schoolId));
    const configRows = await db.select().from(portalItems).where(eq(portalItems.schoolId, schoolId));
    const config = configRows.find((item) => item.kind === "config");
    if (config) {
      await db.update(portalItems).set({ content: subtitle, updatedAt: new Date().toISOString() }).where(eq(portalItems.id, config.id));
    } else {
      await db.insert(portalItems).values({ schoolId, kind: "config", title: "Perfil da escola", content: subtitle, metadata: "{}", sortOrder: 0 });
    }
    return Response.json({ school: { id: schoolId, name }, subtitle, ok: true });
  }

  if (body.action === "login") {
    const codeHash = await sha256(body.code ?? "");
    const [school] = await db.select({ id: schools.id, name: schools.name }).from(schools).where(eq(schools.codeHash, codeHash)).limit(1);
    if (!school) return Response.json({ error: "Código da escola inválido." }, { status: 401 });
    const token = crypto.randomUUID() + crypto.randomUUID();
    const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toISOString();
    await db.insert(schoolSessions).values({ token, schoolId: school.id, expiresAt });
    return Response.json(
      { role: "school", school },
      { headers: withCookies(cookie(token, 30 * 24 * 60 * 60), adminCookie("", 0)) },
    );
  }

  if (body.action === "logout") {
    const token = readCookie(request, SESSION_COOKIE);
    if (token) await db.delete(schoolSessions).where(eq(schoolSessions.token, token));
    return Response.json({ ok: true }, { headers: withCookies(cookie("", 0), adminCookie("", 0)) });
  }

  return Response.json({ error: "Ação inválida." }, { status: 400 });
}

```

## app/api/school-auth.ts

```ts
import { and, eq, gt } from "drizzle-orm";
import { getDb } from "../../db";
import { schoolSessions, schools } from "../../db/schema";

export const SESSION_COOKIE = "pop_school_session";
export const ADMIN_COOKIE = "pop_admin_session";

export async function isAdminRequest(request: Request) {
  const hostname = new URL(request.url).hostname;
  if (hostname === "terminal.local" || hostname === "localhost") return true;
  const token = readCookie(request, ADMIN_COOKIE);
  return Boolean(token && process.env.ADMIN_SESSION_TOKEN && token === process.env.ADMIN_SESSION_TOKEN);
}

export function readCookie(request: Request, name: string) {
  const cookies = request.headers.get("cookie") ?? "";
  for (const part of cookies.split(";")) {
    const [key, ...value] = part.trim().split("=");
    if (key === name) return decodeURIComponent(value.join("="));
  }
  return "";
}

export async function sha256(value: string) {
  const bytes = new TextEncoder().encode(value.trim().toUpperCase());
  const digest = await crypto.subtle.digest("SHA-256", bytes);
  return [...new Uint8Array(digest)].map((byte) => byte.toString(16).padStart(2, "0")).join("");
}

export function newSchoolCode() {
  const alphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
  const bytes = crypto.getRandomValues(new Uint8Array(7));
  const suffix = [...bytes].map((byte) => alphabet[byte % alphabet.length]).join("");
  return `POP-${suffix}`;
}

export async function getSchoolSession(request: Request) {
  const token = readCookie(request, SESSION_COOKIE);
  if (!token) return null;
  const db = await getDb();
  const [result] = await db
    .select({
      id: schools.id,
      name: schools.name,
      expiresAt: schoolSessions.expiresAt,
    })
    .from(schoolSessions)
    .innerJoin(schools, eq(schools.id, schoolSessions.schoolId))
    .where(and(eq(schoolSessions.token, token), gt(schoolSessions.expiresAt, new Date().toISOString())))
    .limit(1);
  return result ?? null;
}

export async function resolveSchoolId(request: Request) {
  const url = new URL(request.url);
  if (await isAdminRequest(request)) {
    const selected = url.searchParams.get("schoolId")?.trim();
    return selected || null;
  }
  return (await getSchoolSession(request))?.id ?? null;
}

```

## app/api/settings/route.ts

```ts
import { eq } from "drizzle-orm";
import { getDb } from "../../../db";
import { portalSettings } from "../../../db/schema";
import { isAdminRequest } from "../school-auth";

export const dynamic = "force-dynamic";

export const defaults = {
  brandName: "POP CHICLÉ PSIEE",
  welcomeTitle: "Nossa escola em movimento.",
  welcomeText: "Agenda, trabalhos, registros, materiais e comunicação em um espaço exclusivo.",
  assessorName: "Dynjara Costa",
  assessorTitle: "Assessoria Pedagógica Especializada",
  primary: "#5b76e8",
  pink: "#ff4f9b",
  yellow: "#ffd84a",
  background: "#eef1f8",
};

export async function GET() {
  const db = await getDb();
  const rows = await db.select().from(portalSettings);
  return Response.json({ settings: { ...defaults, ...Object.fromEntries(rows.map((row) => [row.key, row.value])) } });
}

export async function PATCH(request: Request) {
  if (!(await isAdminRequest(request))) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
  const body = await request.json() as Record<string, unknown>;
  const db = await getDb();
  const allowed = new Set(Object.keys(defaults));
  for (const [key, raw] of Object.entries(body)) {
    if (!allowed.has(key) || typeof raw !== "string") continue;
    const value = raw.trim().slice(0, 1000);
    await db.insert(portalSettings).values({ key, value }).onConflictDoUpdate({
      target: portalSettings.key, set: { value, updatedAt: new Date().toISOString() },
    });
  }
  return Response.json({ ok: true });
}

```

## app/api/portal-items/route.ts

```ts
import { and, desc, eq } from "drizzle-orm";
import { getDb } from "../../../db";
import { portalItems, schools } from "../../../db/schema";
import { isAdminRequest, readCookie, resolveSchoolId, SESSION_COOKIE } from "../school-auth";

export const dynamic = "force-dynamic";

const ADMIN_KINDS = new Set(["agenda", "trabalho", "orientacao", "depoimento", "pasta", "material", "galeria"]);

async function context(request: Request) {
  const schoolId = await resolveSchoolId(request);
  if (!schoolId) return null;
  const db = await getDb();
  const [school] = await db.select({ id: schools.id, name: schools.name }).from(schools).where(eq(schools.id, schoolId)).limit(1);
  return school ? { db, school, admin: await isAdminRequest(request) } : null;
}

export async function GET(request: Request) {
  const ctx = await context(request);
  if (!ctx) return Response.json({ error: "Acesso da escola necessário." }, { status: 401 });
  const rows = await ctx.db
    .select()
    .from(portalItems)
    .where(eq(portalItems.schoolId, ctx.school.id))
    .orderBy(desc(portalItems.createdAt));
  const parsed = rows.map((row) => ({ ...row, metadata: JSON.parse(row.metadata || "{}") }));
  const requestedActor = new URL(request.url).searchParams.get("actor")?.slice(0, 80);
  const actor = ctx.admin ? "admin" : requestedActor || readCookie(request, SESSION_COOKIE);
  const likes = parsed.filter((item) => item.kind === "curtida");
  const comments = parsed.filter((item) => item.kind === "comentario");
  const content = parsed.filter((item) => !["curtida", "comentario"].includes(item.kind));
  for (const item of content) {
    if (!item.metadata.shareToken && !["config", "perfil"].includes(item.kind)) {
      item.metadata.shareToken = crypto.randomUUID();
      await ctx.db.update(portalItems).set({ metadata: JSON.stringify(item.metadata) })
        .where(and(eq(portalItems.id, item.id), eq(portalItems.schoolId, ctx.school.id)));
    }
  }
  return Response.json({
    school: ctx.school,
    admin: ctx.admin,
    items: content.map((item) => ({
      ...item,
      likes: likes.filter((like) => Number(like.metadata.parentId) === item.id).length,
      liked: likes.some((like) => Number(like.metadata.parentId) === item.id && like.metadata.actor === actor),
      comments: comments
        .filter((comment) => Number(comment.metadata.parentId) === item.id)
        .map((comment) => ({ id: comment.id, author: comment.title, content: comment.content, createdAt: comment.createdAt })),
    })),
  });
}

export async function POST(request: Request) {
  const ctx = await context(request);
  if (!ctx) return Response.json({ error: "Acesso da escola necessário." }, { status: 401 });
  const body = (await request.json()) as { action?: string; parentId?: number; actor?: string; kind?: string; title?: string; content?: string; metadata?: unknown };
  if (body.action === "toggle-like" && body.parentId) {
    const [parent] = await ctx.db.select({ id: portalItems.id }).from(portalItems)
      .where(and(eq(portalItems.id, body.parentId), eq(portalItems.schoolId, ctx.school.id))).limit(1);
    if (!parent) return Response.json({ error: "Publicação não encontrada." }, { status: 404 });
    const actor = ctx.admin ? "admin" : body.actor?.trim().slice(0, 80) || readCookie(request, SESSION_COOKIE);
    const likes = await ctx.db.select().from(portalItems)
      .where(and(eq(portalItems.schoolId, ctx.school.id), eq(portalItems.kind, "curtida")));
    const existing = likes.find((like) => {
      const metadata = JSON.parse(like.metadata || "{}");
      return Number(metadata.parentId) === body.parentId && metadata.actor === actor;
    });
    if (existing) {
      await ctx.db.delete(portalItems).where(eq(portalItems.id, existing.id));
      return Response.json({ liked: false });
    }
    await ctx.db.insert(portalItems).values({
      schoolId: ctx.school.id, kind: "curtida", title: "Curtida", content: "",
      metadata: JSON.stringify({ parentId: body.parentId, actor }), sortOrder: Date.now(),
    });
    return Response.json({ liked: true });
  }
  const kind = body.kind?.trim() ?? "";
  const title = body.title?.trim() ?? "";
  if (!title || (!ADMIN_KINDS.has(kind) && !["recado", "comentario"].includes(kind))) {
    return Response.json({ error: "Preencha os dados da publicação." }, { status: 400 });
  }
  if (!ctx.admin && !["recado", "depoimento", "comentario"].includes(kind)) {
    return Response.json({ error: "Somente a assessoria pode publicar este conteúdo." }, { status: 403 });
  }
  const metadata = {
    ...(typeof body.metadata === "object" && body.metadata ? body.metadata : {}),
    status: kind === "comentario" ? "publicado" : ctx.admin ? "publicado" : "pendente",
  };
  const [created] = await ctx.db
    .insert(portalItems)
    .values({
      schoolId: ctx.school.id,
      kind,
      title,
      content: body.content?.trim() ?? "",
      metadata: JSON.stringify(metadata),
      sortOrder: Date.now(),
    })
    .returning();
  return Response.json({ item: { ...created, metadata } }, { status: 201 });
}

export async function PATCH(request: Request) {
  const ctx = await context(request);
  if (!ctx?.admin) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
  const body = (await request.json()) as { id?: number; title?: string; content?: string; metadata?: unknown };
  if (!body.id) return Response.json({ error: "Publicação não informada." }, { status: 400 });
  const [current] = await ctx.db
    .select()
    .from(portalItems)
    .where(and(eq(portalItems.id, body.id), eq(portalItems.schoolId, ctx.school.id)))
    .limit(1);
  if (!current) return Response.json({ error: "Publicação não encontrada." }, { status: 404 });
  const metadata = {
    ...JSON.parse(current.metadata || "{}"),
    ...(typeof body.metadata === "object" && body.metadata ? body.metadata : {}),
  };
  await ctx.db
    .update(portalItems)
    .set({
      title: body.title?.trim() || current.title,
      content: body.content === undefined ? current.content : body.content.trim(),
      metadata: JSON.stringify(metadata),
      updatedAt: new Date().toISOString(),
    })
    .where(and(eq(portalItems.id, body.id), eq(portalItems.schoolId, ctx.school.id)));
  return Response.json({ ok: true });
}

export async function DELETE(request: Request) {
  const ctx = await context(request);
  if (!ctx?.admin) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
  const id = Number(new URL(request.url).searchParams.get("id"));
  if (!id) return Response.json({ error: "Publicação não informada." }, { status: 400 });
  await ctx.db.delete(portalItems).where(and(eq(portalItems.id, id), eq(portalItems.schoolId, ctx.school.id)));
  return Response.json({ ok: true });
}

```

## app/api/portal-upload/route.ts

```ts
import { and, eq } from "drizzle-orm";
import { getDb } from "../../../db";
import { portalItems, schools } from "../../../db/schema";
import { isAdminRequest, resolveSchoolId } from "../school-auth";
import { createSignedUpload, deleteFile } from "../../../lib/storage";

export const dynamic = "force-dynamic";

async function context(request: Request) {
  const schoolId = await resolveSchoolId(request);
  if (!schoolId) return null;
  const db = await getDb();
  const [school] = await db.select({ id: schools.id }).from(schools).where(eq(schools.id, schoolId)).limit(1);
  return school ? { db, school, admin: await isAdminRequest(request) } : null;
}

export async function POST(request: Request) {
  const ctx = await context(request);
  if (!ctx) return Response.json({ error: "Acesso necessário." }, { status: 401 });
  const body = await request.json() as Record<string, unknown>;
  const action = String(body.action || "");

  if (action === "init") {
    const filename = String(body.filename || "arquivo").replace(/[^a-zA-Z0-9._-]/g, "_");
    const contentType = String(body.contentType || "application/octet-stream");
    const key = `${ctx.school.id}/${crypto.randomUUID()}-${filename}`;
    const signed = await createSignedUpload(key);
    return Response.json({ key, path: signed.path, signedUrl: signed.signedUrl, token: crypto.randomUUID(), filename, contentType });
  }

  if (action === "publish") {
    const kind = String(body.kind || "");
    if (!["galeria", "material", "perfil", "trabalho", "orientacao", "recado", "depoimento"].includes(kind)) {
      return Response.json({ error: "Tipo de publicação inválido." }, { status: 400 });
    }
    if (!ctx.admin && !["recado", "depoimento"].includes(kind)) {
      return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
    }
    const files = Array.isArray(body.files) ? body.files as Array<Record<string, unknown>> : [];
    if (!files.length || files.some((file) => !String(file.key || "").startsWith(`${ctx.school.id}/`))) {
      return Response.json({ error: "Arquivos inválidos." }, { status: 400 });
    }
    const first = files[0];
    const metadata = {
      ...first, files, folder: String(body.folder || ""), date: String(body.date || ""),
      category: String(body.category || ""), responsible: String(body.responsible || ""),
      link: String(body.link || ""), workStatus: String(body.workStatus || "Em andamento"),
      audience: String(body.audience || "Equipe escolar"), priority: String(body.priority || "Normal"),
      status: ctx.admin ? "publicado" : "pendente",
    };
    const folder = String(body.folder || "");
    if (kind === "perfil") {
      const previous = await ctx.db.select().from(portalItems)
        .where(and(eq(portalItems.schoolId, ctx.school.id), eq(portalItems.kind, "perfil")));
      for (const item of previous) {
        const old = JSON.parse(item.metadata || "{}") as { key?: string; files?: Array<{ key: string }> };
        for (const stored of old.files || (old.key ? [{ key: old.key }] : [])) await deleteFile(stored.key);
      }
      await ctx.db.delete(portalItems).where(and(eq(portalItems.schoolId, ctx.school.id), eq(portalItems.kind, "perfil")));
    }
    if (kind === "material" && folder) {
      const folders = await ctx.db.select().from(portalItems)
        .where(and(eq(portalItems.schoolId, ctx.school.id), eq(portalItems.kind, "pasta")));
      if (!folders.some((item) => item.title === folder)) {
        await ctx.db.insert(portalItems).values({ schoolId: ctx.school.id, kind: "pasta", title: folder, content: "", metadata: JSON.stringify({ status: "publicado", link: "" }), sortOrder: Date.now() });
      }
    }
    const [created] = await ctx.db.insert(portalItems).values({
      schoolId: ctx.school.id, kind, title: String(body.title || files[0].filename || "Publicação"),
      content: String(body.content || ""), metadata: JSON.stringify(metadata), sortOrder: Date.now(),
    }).returning();
    return Response.json({ item: { ...created, metadata } }, { status: 201 });
  }

  return Response.json({ error: "Ação inválida." }, { status: 400 });
}

```

## app/api/portal-files/route.ts

```ts
import { and, eq } from "drizzle-orm";
import { getDb } from "../../../db";
import { portalItems, schools } from "../../../db/schema";
import { isAdminRequest, resolveSchoolId } from "../school-auth";
import { deleteFile, downloadFile, uploadFile } from "../../../lib/storage";

export const dynamic = "force-dynamic";

async function context(request: Request) {
  const schoolId = await resolveSchoolId(request);
  if (!schoolId) return null;
  const db = await getDb();
  const [school] = await db.select({ id: schools.id, name: schools.name }).from(schools).where(eq(schools.id, schoolId)).limit(1);
  return school ? { db, school, admin: await isAdminRequest(request) } : null;
}

export async function POST(request: Request) {
  const ctx = await context(request);
  if (!ctx) return Response.json({ error: "Acesso necessário." }, { status: 401 });
  const data = await request.formData();
  const kind = String(data.get("kind") || "");
  const title = String(data.get("title") || "").trim();
  const folder = String(data.get("folder") || "").trim();
  const content = String(data.get("content") || "").trim();
  const multiple = data.getAll("files").filter((entry): entry is File => entry instanceof File && entry.size > 0);
  const single = data.get("file");
  const files = multiple.length ? multiple : single instanceof File && single.size > 0 ? [single] : [];
  if (!["galeria", "material", "perfil", "trabalho", "orientacao", "recado", "depoimento"].includes(kind) || !files.length) {
    return Response.json({ error: "Selecione pelo menos um arquivo válido." }, { status: 400 });
  }
  if (!ctx.admin && !["recado", "depoimento"].includes(kind)) return Response.json({ error: "Acesso administrativo necessário." }, { status: 403 });
  if (files.length > 10) return Response.json({ error: "Envie no máximo 10 arquivos por publicação." }, { status: 400 });
  if (kind === "perfil" && !files[0].type.startsWith("image/")) {
    return Response.json({ error: "A foto da escola precisa ser uma imagem." }, { status: 400 });
  }
  const totalSize = files.reduce((sum, file) => sum + file.size, 0);
  if (files.some((file) => file.size > 1024 * 1024 * 1024) || totalSize > 1024 * 1024 * 1024) {
    return Response.json({ error: "A publicação pode ter até 1 GB no total." }, { status: 413 });
  }
  const storedFiles = [];
  for (const file of files) {
    const id = crypto.randomUUID();
    const key = `${ctx.school.id}/${id}-${file.name.replace(/[^a-zA-Z0-9._-]/g, "_")}`;
    await uploadFile(key, await file.arrayBuffer(), file.type || "application/octet-stream");
    storedFiles.push({ key, token: crypto.randomUUID(), filename: file.name, contentType: file.type || "application/octet-stream", size: file.size });
  }
  const first = storedFiles[0];
  const metadata = {
    ...first, files: storedFiles, folder,
    date: String(data.get("date") || ""), category: String(data.get("category") || ""),
    responsible: String(data.get("responsible") || ""), link: String(data.get("link") || ""),
    workStatus: String(data.get("workStatus") || "Em andamento"),
    audience: String(data.get("audience") || "Equipe escolar"),
    priority: String(data.get("priority") || "Normal"),
    status: ctx.admin ? "publicado" : "pendente",
  };
  if (kind === "perfil") {
    const previous = await ctx.db
      .select()
      .from(portalItems)
      .where(and(eq(portalItems.schoolId, ctx.school.id), eq(portalItems.kind, "perfil")));
    for (const item of previous) {
      const old = JSON.parse(item.metadata || "{}") as { key?: string; files?: Array<{ key: string }> };
      for (const stored of old.files || (old.key ? [{ key: old.key }] : [])) await deleteFile(stored.key);
    }
    await ctx.db.delete(portalItems).where(and(eq(portalItems.schoolId, ctx.school.id), eq(portalItems.kind, "perfil")));
  }
  if (kind === "material" && folder) {
    const [existingFolder] = await ctx.db
      .select({ id: portalItems.id })
      .from(portalItems)
      .where(and(eq(portalItems.schoolId, ctx.school.id), eq(portalItems.kind, "pasta"), eq(portalItems.title, folder)))
      .limit(1);
    if (!existingFolder) {
      await ctx.db.insert(portalItems).values({
        schoolId: ctx.school.id,
        kind: "pasta",
        title: folder,
        content: "",
        metadata: JSON.stringify({ status: "publicado", link: "" }),
        sortOrder: Date.now(),
      });
    }
  }
  const [created] = await ctx.db
    .insert(portalItems)
    .values({
      schoolId: ctx.school.id,
      kind,
      title: title || files[0].name,
      content,
      metadata: JSON.stringify(metadata),
      sortOrder: Date.now(),
    })
    .returning();
  return Response.json({ item: { ...created, metadata } }, { status: 201 });
}

export async function GET(request: Request) {
  const url = new URL(request.url);
  const id = Number(url.searchParams.get("id"));
  const index = Math.max(0, Number(url.searchParams.get("index") || 0));
  const token = url.searchParams.get("token") || "";
  const db = await getDb();
  const [item] = await db
    .select()
    .from(portalItems)
    .where(eq(portalItems.id, id))
    .limit(1);
  if (!item) return new Response("Arquivo não encontrado.", { status: 404 });
  const metadata = JSON.parse(item.metadata || "{}") as { key?: string; token?: string; filename?: string; contentType?: string; files?: Array<{ key: string; token: string; filename: string; contentType: string }> };
  const selected = metadata.files?.[index] || metadata;
  if (!selected.token || token !== selected.token) {
    const ctx = await context(request);
    if (!ctx || ctx.school.id !== item.schoolId) return new Response("Acesso necessário.", { status: 401 });
  }
  if (!selected.key) return new Response("Arquivo não encontrado.", { status: 404 });
  const object = await downloadFile(selected.key);
  if (!object) return new Response("Arquivo não encontrado.", { status: 404 });
  return new Response(object, {
    headers: {
      "content-type": selected.contentType || "application/octet-stream",
      "content-disposition": `inline; filename="${(selected.filename || "arquivo").replace(/"/g, "")}"`,
      "cache-control": "public, max-age=3600",
    },
  });
}

```

## app/api/share-image/route.ts

```ts
import { and, eq } from "drizzle-orm";
import { getDb } from "../../../db";
import { portalItems } from "../../../db/schema";
import { downloadFile } from "../../../lib/storage";

export const dynamic = "force-dynamic";

export async function GET(request: Request) {
  const url = new URL(request.url);
  const id = Number(url.searchParams.get("id"));
  const token = url.searchParams.get("token") || "";
  if (!Number.isInteger(id) || id < 1 || !token) {
    return new Response("Miniatura não encontrada.", { status: 404 });
  }

  const db = await getDb();
  const [item] = await db.select().from(portalItems).where(eq(portalItems.id, id)).limit(1);
  if (!item) return new Response("Miniatura não encontrada.", { status: 404 });

  const metadata = JSON.parse(item.metadata || "{}");
  if (metadata.shareToken !== token) return new Response("Miniatura não encontrada.", { status: 404 });

  let mediaMetadata = metadata;
  let file = mediaMetadata.files?.find((entry: { contentType?: string }) => entry.contentType?.startsWith("image/"))
    || (mediaMetadata.contentType?.startsWith("image/") ? mediaMetadata : null);
  if (!file) {
    const [profile] = await db.select().from(portalItems)
      .where(and(eq(portalItems.schoolId, item.schoolId), eq(portalItems.kind, "perfil"))).limit(1);
    if (profile) {
      mediaMetadata = JSON.parse(profile.metadata || "{}");
      file = mediaMetadata.files?.find((entry: { contentType?: string }) => entry.contentType?.startsWith("image/"))
        || (mediaMetadata.contentType?.startsWith("image/") ? mediaMetadata : null);
    }
  }
  if (!file?.key) return new Response("Miniatura não encontrada.", { status: 404 });

  const object = await downloadFile(file.key);
  if (!object) return new Response("Miniatura não encontrada.", { status: 404 });

  return new Response(object, {
    headers: {
      "content-type": file.contentType || "image/jpeg",
      "content-length": String(object.size),
      "cache-control": "public, max-age=86400",
    },
  });
}

```

## app/api/miniatura/[id]/[token]/route.ts

```ts
import { and, eq } from "drizzle-orm";
import { getDb } from "../../../../../db";
import { portalItems } from "../../../../../db/schema";
import { downloadFile } from "../../../../../lib/storage";

export const dynamic = "force-dynamic";

export async function GET(_request: Request, { params }: { params: Promise<{ id: string; token: string }> }) {
  const { id, token } = await params;
  const db = await getDb();
  const [item] = await db.select().from(portalItems).where(eq(portalItems.id, Number(id))).limit(1);
  if (!item) return new Response("Miniatura não encontrada.", { status: 404 });
  const metadata = JSON.parse(item.metadata || "{}");
  if (!token || metadata.shareToken !== token) return new Response("Miniatura não encontrada.", { status: 404 });
  let mediaMetadata = metadata;
  let file = mediaMetadata.files?.find((entry: { contentType?: string }) => entry.contentType?.startsWith("image/"))
    || (mediaMetadata.contentType?.startsWith("image/") ? mediaMetadata : null);
  if (!file) {
    const [profile] = await db.select().from(portalItems)
      .where(and(eq(portalItems.schoolId, item.schoolId), eq(portalItems.kind, "perfil"))).limit(1);
    if (profile) {
      mediaMetadata = JSON.parse(profile.metadata || "{}");
      file = mediaMetadata.files?.find((entry: { contentType?: string }) => entry.contentType?.startsWith("image/"))
        || (mediaMetadata.contentType?.startsWith("image/") ? mediaMetadata : null);
    }
  }
  if (!file?.key) return new Response("Miniatura não encontrada.", { status: 404 });
  const object = await downloadFile(file.key);
  if (!object) return new Response("Miniatura não encontrada.", { status: 404 });
  return new Response(object, {
    headers: {
      "content-type": file.contentType || "image/jpeg",
      "content-length": String(object.size),
      "cache-control": "public, max-age=86400",
    },
  });
}

```

## app/publicacao/[id]/page.tsx

```tsx
import type { Metadata } from "next";
import { and, eq } from "drizzle-orm";
import { getDb } from "../../../db";
import { portalItems, schools } from "../../../db/schema";

const BASE = "https://plataforma-escolar-pop-chicle.popchiclepsiee.chatgpt.site";

async function sharedPost(id: number, token: string) {
  const db = await getDb();
  const [item] = await db.select().from(portalItems).where(eq(portalItems.id, id)).limit(1);
  if (!item) return null;
  const metadata = JSON.parse(item.metadata || "{}");
  if (!token || metadata.shareToken !== token) return null;
  const [school] = await db.select({ name: schools.name }).from(schools).where(eq(schools.id, item.schoolId)).limit(1);
  let imageMeta = metadata;
  let imageFile = imageMeta.files?.find((entry: { contentType?: string }) => entry.contentType?.startsWith("image/"))
    || (imageMeta.contentType?.startsWith("image/") ? imageMeta : null);
  if (!imageFile) {
    const [profile] = await db.select().from(portalItems)
      .where(and(eq(portalItems.schoolId, item.schoolId), eq(portalItems.kind, "perfil"))).limit(1);
    if (profile) {
      imageMeta = JSON.parse(profile.metadata || "{}");
      imageFile = imageMeta.files?.find((entry: { contentType?: string }) => entry.contentType?.startsWith("image/"))
        || (imageMeta.contentType?.startsWith("image/") ? imageMeta : null);
    }
  }
  const image = imageFile?.key
    ? `${BASE}/api/share-image?id=${item.id}&token=${encodeURIComponent(token)}`
    : undefined;
  return { item, metadata, school, image };
}

export async function generateMetadata({ params, searchParams }: { params: Promise<{ id: string }>; searchParams: Promise<{ token?: string }> }): Promise<Metadata> {
  const { id } = await params;
  const { token = "" } = await searchParams;
  const post = await sharedPost(Number(id), token);
  if (!post) return { title: "Publicação não encontrada" };
  const description = post.item.content?.slice(0, 180) || `Publicação de ${post.school?.name || "escola assessorada"}`;
  return {
    title: `${post.item.title} | POP CHICLÉ PSIEE`,
    description,
    openGraph: {
      title: post.item.title,
      description,
      type: "article",
      url: `${BASE}/publicacao/${id}?token=${encodeURIComponent(token)}`,
      images: post.image ? [{ url: post.image, alt: post.item.title, width: 1200, height: 630, type: "image/jpeg" }] : [],
    },
    twitter: { card: "summary_large_image", title: post.item.title, description, images: post.image ? [post.image] : [] },
  };
}

export default async function SharedPublication({ params, searchParams }: { params: Promise<{ id: string }>; searchParams: Promise<{ token?: string }> }) {
  const { id } = await params;
  const { token = "" } = await searchParams;
  const post = await sharedPost(Number(id), token);
  if (!post) return <main className="shared-page"><article><h1>Publicação não encontrada</h1><a href="/">Voltar para a plataforma</a></article></main>;
  const file = post.metadata.files?.[0] || post.metadata;
  const media = file.key && file.token ? `${BASE}/api/portal-files?id=${post.item.id}&index=0&token=${encodeURIComponent(file.token)}` : null;
  return <main className="shared-page"><article><header><b>POP CHICLÉ PSIEE</b><span>{post.school?.name}</span></header>{media && file.contentType?.startsWith("image/") ? <img src={media} alt={post.item.title} /> : media && file.contentType?.startsWith("video/") ? <video src={`${media}#t=0.001`} controls playsInline preload="auto" /> : null}<div><small>PUBLICAÇÃO DA ESCOLA ASSESSORADA</small><h1>{post.item.title}</h1><p>{post.item.content}</p><a href="/">Acessar a plataforma</a></div></article></main>;
}

```

## db/index.ts

```ts
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import * as schema from "./schema";

export async function getDb() {
  const url = process.env.DATABASE_URL;
  if (!url) throw new Error("DATABASE_URL não configurada.");
  const client = postgres(url, { prepare: false, max: 5 });
  return drizzle(client, { schema });
}

```

## db/schema.ts

```ts
import { sql } from "drizzle-orm";
import { index, integer, pgTable, serial, text, uniqueIndex } from "drizzle-orm/pg-core";

export const portalSettings = pgTable("portal_settings", {
  key: text("key").primaryKey(),
  value: text("value").notNull(),
  updatedAt: text("updated_at").notNull().default(sql`CURRENT_TIMESTAMP`),
});

export const portalItems = pgTable("portal_items", {
  id: serial("id").primaryKey(),
  schoolId: text("school_id").notNull().default("default"),
  kind: text("kind").notNull(),
  title: text("title").notNull(),
  content: text("content").notNull().default(""),
  metadata: text("metadata").notNull().default("{}"),
  sortOrder: integer("sort_order").notNull().default(0),
  createdAt: text("created_at").notNull().default(sql`CURRENT_TIMESTAMP`),
  updatedAt: text("updated_at").notNull().default(sql`CURRENT_TIMESTAMP`),
}, (table) => [
  index("portal_items_school_kind_idx").on(table.schoolId, table.kind),
]);

export const portalSchools = pgTable("portal_schools", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  userEmail: text("user_email").notNull(),
  createdAt: text("created_at").notNull().default(sql`CURRENT_TIMESTAMP`),
}, (table) => [
  uniqueIndex("portal_schools_user_email_idx").on(table.userEmail),
]);

export const schools = pgTable("schools", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  codeHash: text("code_hash").notNull().unique(),
  accessCode: text("access_code"),
  createdAt: text("created_at").notNull().default(sql`CURRENT_TIMESTAMP`),
  updatedAt: text("updated_at").notNull().default(sql`CURRENT_TIMESTAMP`),
});

export const schoolSessions = pgTable("school_sessions", {
  token: text("token").primaryKey(),
  schoolId: text("school_id").notNull(),
  expiresAt: text("expires_at").notNull(),
  createdAt: text("created_at").notNull().default(sql`CURRENT_TIMESTAMP`),
});

export const schoolPortals = pgTable("school_portals", {
  schoolId: text("school_id").primaryKey(),
  dataJson: text("data_json").notNull().default("{}"),
  htmlKey: text("html_key"),
  updatedAt: text("updated_at").notNull().default(sql`CURRENT_TIMESTAMP`),
});

```

## lib/storage.ts

```ts
import { createClient } from "@supabase/supabase-js";

export const STORAGE_BUCKET = "school-files";

function admin() {
  const url = process.env.SUPABASE_URL || process.env.NEXT_PUBLIC_SUPABASE_URL;
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY;
  if (!url || !key) throw new Error("Supabase não configurado.");
  return createClient(url, key, { auth: { persistSession: false } });
}

export async function createSignedUpload(path: string) {
  const { data, error } = await admin().storage.from(STORAGE_BUCKET).createSignedUploadUrl(path);
  if (error) throw error;
  return data;
}

export async function uploadFile(path: string, data: ArrayBuffer, contentType: string) {
  const { error } = await admin().storage.from(STORAGE_BUCKET).upload(path, data, {
    contentType,
    upsert: false,
  });
  if (error) throw error;
}

export async function downloadFile(path: string) {
  const { data, error } = await admin().storage.from(STORAGE_BUCKET).download(path);
  if (error || !data) return null;
  return data;
}

export async function deleteFile(path: string) {
  const { error } = await admin().storage.from(STORAGE_BUCKET).remove([path]);
  if (error) throw error;
}

```

## supabase/schema.sql

```sql
create table if not exists portal_settings (
  key text primary key,
  value text not null,
  updated_at text not null default current_timestamp::text
);

create table if not exists portal_items (
  id serial primary key,
  school_id text not null default 'default',
  kind text not null,
  title text not null,
  content text not null default '',
  metadata text not null default '{}',
  sort_order integer not null default 0,
  created_at text not null default current_timestamp::text,
  updated_at text not null default current_timestamp::text
);
create index if not exists portal_items_school_kind_idx on portal_items (school_id, kind);

create table if not exists portal_schools (
  id text primary key,
  name text not null,
  user_email text not null unique,
  created_at text not null default current_timestamp::text
);

create table if not exists schools (
  id text primary key,
  name text not null,
  code_hash text not null unique,
  access_code text,
  created_at text not null default current_timestamp::text,
  updated_at text not null default current_timestamp::text
);

create table if not exists school_sessions (
  token text primary key,
  school_id text not null,
  expires_at text not null,
  created_at text not null default current_timestamp::text
);

create table if not exists school_portals (
  school_id text primary key,
  data_json text not null default '{}',
  html_key text,
  updated_at text not null default current_timestamp::text
);

insert into storage.buckets (id, name, public, file_size_limit)
values ('school-files', 'school-files', false, 1073741824)
on conflict (id) do update set file_size_limit = 1073741824;

```

## next.config.ts

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  /* config options here */
};

export default nextConfig;

```

## netlify.toml

```toml
[build]
  command = "npm run build"
  publish = ".next"

[build.environment]
  NODE_VERSION = "20"

```

## package.json

```json
{
  "name": "plataforma-escolar-pop-chicle",
  "version": "0.1.0",
  "private": true,
  "engines": {
    "node": ">=20.0.0"
  },
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "test": "npm run build",
    "lint": "eslint . --ignore-pattern .next",
    "db:generate": "drizzle-kit generate"
  },
  "dependencies": {
    "drizzle-orm": "0.45.2",
    "@supabase/supabase-js": "^2.57.4",
    "next": "16.2.6",
    "pdfjs-dist": "^5.4.530",
    "react": "19.2.6",
    "react-dom": "19.2.6",
    "postgres": "^3.4.7"
  },
  "devDependencies": {
    "@tailwindcss/postcss": "4.2.1",
    "@types/node": "22.19.19",
    "@types/react": "19.2.14",
    "@types/react-dom": "19.2.3",
    "drizzle-kit": "0.31.10",
    "eslint": "9.39.4",
    "eslint-config-next": "16.2.6",
    "tailwindcss": "4.2.1",
    "typescript": "5.9.3",
    "react-server-dom-webpack": "19.2.6"
  },
  "type": "module",
  "displayName": "Plataforma Escolar POP CHICLÉ"
}

```

## .env.example

```example
DATABASE_URL=postgresql://postgres.SEU_PROJETO:SUA_SENHA@aws-0-SUA_REGIAO.pooler.supabase.com:6543/postgres
SUPABASE_URL=https://SEU_PROJETO.supabase.co
NEXT_PUBLIC_SUPABASE_URL=https://SEU_PROJETO.supabase.co
SUPABASE_SERVICE_ROLE_KEY=COLE_A_SERVICE_ROLE_KEY
ADMIN_USERNAME=seu_usuario
ADMIN_PASSWORD=crie_uma_senha_forte
ADMIN_SESSION_TOKEN=crie_um_token_longo_e_aleatorio

```

## LEIA-PRIMEIRO.md

```md
# Publicação no Netlify

Este pacote é uma cópia independente do portal. Ele não altera o site atualmente publicado.

## O que você precisa

- Uma conta no Netlify.
- Uma conta no Supabase para guardar escolas, publicações, fotos, PDFs e vídeos.

O Netlify hospeda o site. O Supabase guarda os dados e arquivos. Isso é necessário porque o portal é dinâmico; um HTML isolado não preserva publicações entre navegadores.

## 1. Preparar o Supabase

1. Crie um projeto em https://supabase.com/dashboard.
2. Abra **SQL Editor**, clique em **New query**, copie todo o arquivo `supabase/schema.sql` e execute.
3. Em **Project Settings > API**, copie:
   - Project URL
   - `service_role` key
4. Em **Project Settings > Database**, copie a **Connection string** do modo Transaction pooler.
5. Em **Storage > school-files > Configuration**, confirme o limite desejado. O arquivo SQL solicita 1 GB, mas o limite efetivo depende do seu plano do Supabase.

## 2. Colocar no GitHub

1. Crie um repositório vazio no GitHub.
2. Extraia este ZIP.
3. Envie todos os arquivos da pasta extraída para o repositório, incluindo `netlify.toml`.

## 3. Publicar no Netlify

1. No Netlify, clique em **Add new project > Import an existing project**.
2. Escolha o repositório do GitHub.
3. O Netlify reconhecerá:
   - Build command: `npm run build`
   - Publish directory: `.next`
4. Antes de publicar, abra **Environment variables** e crie:

| Variável | Valor |
| --- | --- |
| `DATABASE_URL` | Connection string do Supabase |
| `SUPABASE_URL` | Project URL do Supabase |
| `NEXT_PUBLIC_SUPABASE_URL` | A mesma Project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Chave `service_role` do Supabase |
| `ADMIN_USERNAME` | Seu usuário de administração |
| `ADMIN_PASSWORD` | Sua senha de administração |
| `ADMIN_SESSION_TOKEN` | Uma sequência aleatória longa, com pelo menos 40 caracteres |

5. Clique em **Deploy**.

## 4. Entrar

- Abra o endereço gerado pelo Netlify.
- Entre como administradora com `ADMIN_USERNAME` e `ADMIN_PASSWORD`.
- Crie as escolas e seus códigos normalmente.

## Importante

- Não publique nem envie o arquivo `.env` para o GitHub. O pacote contém somente `.env.example`, sem senhas.
- O Netlify e o Supabase possuem limites de plano. Nenhuma hospedagem legítima pode prometer armazenamento e funcionamento ilimitados para sempre.
- Os dados do site atual não migram automaticamente, pois pertencem ao banco da hospedagem atual. Este pacote começa com um banco novo.
- Depois de publicar, use o domínio do Netlify como endereço oficial. Assim, cancelar ou mudar o plano do ChatGPT não altera essa nova cópia.

```

