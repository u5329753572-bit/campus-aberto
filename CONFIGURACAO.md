# 🚀 Guia de Configuração — Comunidade UAb

## O que tens
- `index.html`  → Página de login e registo
- `app.html`    → Área de membros (UCs e documentos)
- `admin.html`  → Painel de administração (só tu)

---

## PASSO 1 — Criar projeto Firebase (gratuito)

1. Vai a https://console.firebase.google.com
2. Clica em **"Adicionar projeto"**
3. Dá um nome (ex: `uab-comunidade`)
4. Desativa Google Analytics (não precisas)
5. Clica **"Criar projeto"**

---

## PASSO 2 — Ativar Authentication

1. No menu lateral, clica **Authentication**
2. Clica **"Começar"**
3. Em "Método de login", ativa **"E-mail/password"**
4. Guarda

---

## PASSO 3 — Ativar Firestore (base de dados)

1. No menu lateral, clica **Firestore Database**
2. Clica **"Criar base de dados"**
3. Escolhe **"Modo de produção"**
4. Escolhe região: `europe-west1` (Bélgica — mais próxima)
5. Clica **"Concluído"**

### Regras de segurança do Firestore
Vai a **Firestore → Regras** e substitui pelo seguinte:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() {
      return request.auth != null;
    }

    function isAdmin() {
      return signedIn() &&
        get(/databases/$(database)/documents/utilizadores/$(request.auth.uid)).data.admin == true;
    }

    function isActive() {
      return signedIn() &&
        get(/databases/$(database)/documents/utilizadores/$(request.auth.uid)).data.estado == 'ativo';
    }

    // O utilizador cria apenas o seu perfil pendente; só o admin o pode alterar.
    match /utilizadores/{uid} {
      allow create: if signedIn() && request.auth.uid == uid
        && request.resource.data.keys().hasAll(['nome', 'email', 'tel', 'codigo', 'estado', 'criadoEm'])
        && request.resource.data.keys().hasOnly(['nome', 'email', 'tel', 'codigo', 'estado', 'criadoEm'])
        && request.resource.data.estado == 'pendente'
        && request.resource.data.email == request.auth.token.email;
      allow read: if signedIn() && (request.auth.uid == uid || isAdmin());
      allow update, delete: if isAdmin();
    }

    // UCs e documentos: só utilizadores ativos leem; admin escreve
    match /ucs/{ucId} {
      allow read: if isActive();
      allow write: if isAdmin();

      match /documentos/{docId} {
        allow read: if isActive();
        allow write: if isAdmin();
      }
    }

    // Downloads: um utilizador ativo só cria registos associados ao próprio UID.
    match /downloads/{dlId} {
      allow create: if isActive()
        && request.resource.data.uid == request.auth.uid
        && request.resource.data.keys().hasOnly([
          'uid', 'nome', 'email', 'codigoUser', 'codigoDl',
          'docId', 'docNome', 'ucId', 'data'
        ]);
      allow read: if isAdmin();
    }
  }
}
```

---

## PASSO 4 — Preparar o alojamento dos PDFs

Nesta versão, os documentos são lidos através de um URL direto e o Firebase Storage não é utilizado. Coloca os PDFs na pasta `docs/` do repositório e, no painel de administração, usa o endereço do botão **Raw** de cada ficheiro.

> Não publiques documentos que não tenhas autorização para partilhar. Um repositório público torna os ficheiros acessíveis a quem conhecer o URL, mesmo que a interface do CampusAberto exija autenticação.

---

## PASSO 5 — Obter as credenciais Firebase

1. Vai a **Configurações do projeto** (ícone ⚙️ no menu lateral)
2. Em "As tuas apps", clica em **"</ > Web"**
3. Dá o nome `uab-web` e clica **"Registar app"**
4. Copia o objeto `firebaseConfig` que aparece

---

## PASSO 6 — Inserir as credenciais nos ficheiros

Nos 3 ficheiros (`index.html`, `app.html`, `admin.html`), substitui:

```javascript
const firebaseConfig = {
  apiKey: "SUBSTITUI_API_KEY",
  authDomain: "SUBSTITUI.firebaseapp.com",
  projectId: "SUBSTITUI_PROJECT_ID",
  storageBucket: "SUBSTITUI.appspot.com",
  messagingSenderId: "SUBSTITUI",
  appId: "SUBSTITUI"
};
```

...pelos valores reais que copiaste do Firebase.

---

## PASSO 7 — Publicar no GitHub Pages (gratuito)

1. Vai a https://github.com e cria conta (se não tens)
2. Cria um novo repositório: `uab-comunidade` (privado ou público — para GitHub Pages gratuito tem de ser público)
3. Faz upload dos 3 ficheiros HTML
4. Vai a **Settings → Pages**
5. Em "Source", escolhe `main` e pasta `/root`
6. Clica **Save**
7. O site fica disponível em: `https://SEU-UTILIZADOR.github.io/uab-comunidade/`

---

## PASSO 8 — Criar a tua conta de admin

1. Abre o site e regista-te normalmente com o teu email
2. Vai ao **Firebase Console → Firestore → utilizadores**
3. Encontra o teu documento (pelo email)
4. Edita dois campos:
   - `estado` → `ativo`
   - `admin`  → `true`
5. Guarda

A partir daí, acedes a `admin.html` e geres tudo a partir da interface.

---

## Fluxo de utilização diário

```
1. Alguém regista-se no site
2. Tu recebes notificação (ver nota abaixo)
3. Abres admin.html → Utilizadores → Clicas "Ativar"
4. A pessoa entra normalmente

Para adicionar documentos:
admin.html → Documentos → Seleciona UC → Faz upload do PDF → Guarda
```

### Nota sobre notificações de novos registos
O Firebase não envia email automático quando alguém se regista.
Opções:
- Verificas o painel admin regularmente
- Ou configuras **Firebase Functions** (mais avançado) para enviar email automático

---

## Resumo de custos

| Serviço | Plano gratuito |
|---|---|
| Firebase Auth | Ilimitado |
| Firestore | 1 GB dados, 50k leituras/dia |
| Firebase Storage | 5 GB armazenamento |
| GitHub Pages | Ilimitado |
| **Total** | **0€/mês** |

Para uma comunidade estudantil estes limites são mais do que suficientes.
