# 🎨 Manual de Distribuição & Desenvolvimento PWA
### *Produzido por Vincent AI Studios — Edição 2026*

> [!NOTE]
> Este documento é um guia completo em dois volumes: **Volume I** para o usuário final que deseja instalar o app, e **Volume II** para o desenvolvedor que deseja replicar essa tecnologia em seu próprio projeto.

---

# 📱 Volume I: Para o Usuário Final

## O que é um PWA?

Um **PWA (Progressive Web App)** é um site que se comporta como um aplicativo nativo. Você o instala diretamente do navegador — sem passar por loja de apps — e ele fica na sua tela inicial como qualquer outro app, com ícone, tela cheia e suporte parcial a internet lenta.

**Vantagens sobre apps tradicionais:**

| PWA | App da Loja |
|---|---|
| Instalação instantânea | Download de dezenas de MB |
| Atualiza sozinho | Requer nova versão na loja |
| Ocupa menos de 1 MB | Ocupa centenas de MB |
| Funciona no celular E no PC | Versão separada para cada plataforma |

---

## 🚀 Como Instalar: Passo a Passo

### Android (Google Chrome)

1. Abra o Google Chrome e acesse **`vincent-vangogh.web.app`**
2. Aguarde a página carregar completamente
3. Toque no menu **⋮** (três pontos) no canto superior direito
4. Toque em **"Instalar aplicativo"** ou **"Adicionar à tela inicial"**
5. Confirme tocando em **"Instalar"** na janela que aparecer
6. O ícone do Vincent aparecerá na sua lista de apps ✅

> [!TIP]
> Em alguns Android, o Chrome exibe uma barra inferior com **"+ Adicionar ao início"** automaticamente. Basta tocar nela!

---

### iOS / iPhone (Safari)

O iPhone exige que você use o **Safari** — outros navegadores não suportam instalação de PWA no iOS.

1. Abra o **Safari** (não Chrome, não Firefox)
2. Acesse **`vincent-vangogh.web.app`**
3. Toque no ícone de **Compartilhar** na barra inferior
4. Role a lista até encontrar **"Adicionar à Tela de Início"**
5. Toque em **"Adicionar"** no canto superior direito
6. O ícone aparece na sua tela inicial como um app nativo ✅

> [!IMPORTANT]
> No iOS, o PWA não tem acesso a notificações push. No Android, isso é possível após configuração adicional.

---

### Computador — Windows / Mac (Chrome ou Edge)

1. Abra o Chrome ou Edge e acesse o site
2. Olhe para a **barra de endereços** — no canto direito haverá um ícone de computador com `+`
3. Clique nele e selecione **"Instalar"**
4. O Vincent abre em uma janela dedicada, sem a barra do navegador ✅

---

# 🛠️ Volume II: Para o Desenvolvedor

## Visão Geral: O que você vai construir

Um PWA é composto por **3 peças obrigatórias**:

```
seu-site/
├── manifest.json       ← Identidade do app (nome, ícone, cores)
├── sw.js               ← Service Worker (cache e offline)
├── index.html          ← Sua página + registro do SW
└── assets/
    ├── icon-192.png    ← Ícone obrigatório
    └── icon-512.png    ← Ícone obrigatório (alta resolução)
```

---

## Passo 1: Criar o `manifest.json`

O manifesto é o "RG" do seu app. Salve na **raiz** do seu site:

```json
{
  "name": "Nome Completo do Seu App",
  "short_name": "MeuApp",
  "description": "Uma linha descrevendo o que seu app faz.",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#0f172a",
  "theme_color": "#a855f7",
  "orientation": "portrait",
  "categories": ["productivity", "utilities"],
  "icons": [
    {
      "src": "assets/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "assets/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "assets/icon-maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

**Campos essenciais explicados:**

| Campo | O que define |
|---|---|
| `name` | Nome exibido na tela de instalação |
| `short_name` | Nome embaixo do ícone (máx. ~12 chars) |
| `display` | `standalone` = sem barra do navegador |
| `theme_color` | Cor da barra de status do celular |
| `purpose: maskable` | Ícone que se adapta ao formato do Android |

---

## Passo 2: Criar o `sw.js` (Service Worker)

Este arquivo é o coração do PWA. Salve na **raiz** do site:

```javascript
// Controle de versão — INCREMENTE a cada novo deploy
// para forçar a limpeza do cache antigo em todos os dispositivos
const VERSION = 'v1';
const CACHE_NAME = `meu-app-cache-${VERSION}`;

// Arquivos que serão salvos no cache para acesso offline
const ASSETS_TO_CACHE = [
  '/',
  '/index.html',
  '/css/style.css',
  '/js/main.js',
  '/assets/icon-512.png',
  '/favicon.svg'
];

// INSTALL: Salva os assets no cache quando o SW é instalado
self.addEventListener('install', (event) => {
  self.skipWaiting(); // Força a ativação imediata (sem esperar fechar abas)
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      console.log(`SW [${VERSION}]: Cacheando assets...`);
      // allSettled: não falha se algum asset estiver fora do ar
      return Promise.allSettled(ASSETS_TO_CACHE.map(url => cache.add(url)));
    })
  );
});

// ACTIVATE: Assume o controle e limpa caches de versões antigas
self.addEventListener('activate', (event) => {
  event.waitUntil(
    Promise.all([
      self.clients.claim(), // Assume controle de todas as abas abertas
      caches.keys().then((keys) =>
        Promise.all(
          keys
            .filter(key => key !== CACHE_NAME)
            .map(key => caches.delete(key))
        )
      )
    ])
  );
});

// FETCH: Intercepta requisições (estratégia: Stale-While-Revalidate)
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // CRITICO: Só intercepta o próprio domínio.
  // Ignorar esta regra quebra CDNs, APIs externas e localhost!
  if (url.origin !== self.location.origin) return;
  if (event.request.method !== 'GET') return;

  event.respondWith(
    caches.match(event.request).then((cachedResponse) => {
      const networkFetch = fetch(event.request)
        .then((networkResponse) => {
          // Atualiza o cache SOMENTE para respostas 200 não redirecionadas
          if (networkResponse?.status === 200 && !networkResponse.redirected) {
            const clone = networkResponse.clone();
            caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
          }
          return networkResponse;
        })
        .catch(() => {
          // Sem rede: retorna o cache ou um erro 503
          return cachedResponse || new Response('Offline', { status: 503 });
        });

      // Serve cache imediatamente; atualiza por baixo dos panos
      return cachedResponse || networkFetch;
    })
  );
});
```

---

## Passo 3: Registrar no HTML

Adicione em **todo `<head>`** das suas páginas:

```html
<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#a855f7">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

E adicione este script antes do `</body>`:

```html
<script>
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('/sw.js')
        .then(reg => console.log('PWA: Service Worker registrado!', reg.scope));
    });

    // Recarrega a página automaticamente quando um novo SW é ativado
    let refreshing = false;
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      if (refreshing) return;
      refreshing = true;
      window.location.reload();
    });
  }

  // Captura o prompt de instalação para exibir via botão personalizado
  let installPrompt;
  window.addEventListener('beforeinstallprompt', (e) => {
    e.preventDefault();
    installPrompt = e;
  });

  // Para acionar via botão customizado no seu HTML:
  // document.getElementById('btn-instalar').addEventListener('click', () => {
  //   if (installPrompt) installPrompt.prompt();
  // });
</script>
```

---

## Passo 4: Criar os Ícones

Você precisa de pelo menos 2 ícones PNG:
- `icon-192.png` — 192×192px
- `icon-512.png` — 512×512px
- `icon-maskable-512.png` — 512×512px com margem de segurança de 20% (para Android adaptativo)

> [!TIP]
> Use **[maskable.app](https://maskable.app)** para testar como seu ícone fica no formato adaptativo do Android antes de publicar.

---

## Passo 5: Hospedar com HTTPS

PWAs **exigem HTTPS**. As opções mais rápidas para começar:

| Plataforma | Gratuito | Deploy |
|---|---|---|
| **Firebase Hosting** | Sim | `firebase deploy --only hosting` |
| **Vercel** | Sim | `vercel --prod` |
| **Netlify** | Sim | `netlify deploy --prod` |
| **GitHub Pages** | Sim | Push para `gh-pages` |

---

## 🔄 Ciclo de Atualização Automática

Toda vez que você faz deploy, seus usuários recebem a atualização automaticamente:

```
Você faz deploy
    ↓
Browser detecta novo sw.js
    ↓
Novo Service Worker baixa em background
    ↓
skipWaiting() → novo SW assume controle imediatamente
    ↓
controllerchange → página recarrega automaticamente
    ↓
Usuário vê a versão nova
```

**Regra de ouro:** sempre incremente `VERSION` no `sw.js` em cada deploy:

```javascript
// Antes:
const VERSION = 'v1';

// Após o próximo deploy:
const VERSION = 'v2';
```

---

## 🌍 Publicação na Google Play Store (TWA)

Para colocar seu site na loja oficial do Google, usamos **TWA (Trusted Web Activity)** — tecnologia que empacota seu PWA como app Android nativo.

### Pré-requisitos
- Site com HTTPS e `manifest.json` válido
- Conta no Google Play Console (taxa única de $25)
- Node.js instalado
- JDK 11+ (para assinar o app)

### Instalar o Bubblewrap

```bash
npm install -g @bubblewrap/cli
```

### Inicializar o projeto Android

```bash
bubblewrap init --manifest https://seusite.com/manifest.json
```

O CLI fará perguntas interativas (nome do pacote, versão etc.) e gerará um projeto Android completo.

### Gerar o bundle `.aab` para a loja

```bash
bubblewrap build
```

O arquivo `app-release-bundle.aab` é o que você envia ao Google Play Console.

### Criar o `assetlinks.json` (obrigatório para tela cheia)

Sem este arquivo, o app exibirá a barra de URL do Chrome. Crie o arquivo na URL:
`seusite.com/.well-known/assetlinks.json`

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.seuapp.pacote",
    "sha256_cert_fingerprints": [
      "AA:BB:CC:DD:EE:FF:..."
    ]
  }
}]
```

> [!IMPORTANT]
> O `sha256_cert_fingerprints` é gerado pelo `bubblewrap build`. Copie exatamente como aparece no terminal e suba o arquivo para o Firebase antes de enviar o app para revisão.

### Cronograma Sugerido

| Semana | Ação |
|---|---|
| **1** | Criar conta Google Play Console ($25) |
| **2** | Preparar ícones e screenshots para a loja |
| **3** | `bubblewrap init && build`, criar `assetlinks.json` |
| **4** | Enviar para revisão (3 a 7 dias úteis) |

---

## ✅ Checklist Final do Desenvolvedor

- [ ] `manifest.json` com `name`, `icons`, `start_url` e `display: standalone`
- [ ] Ícones de 192px e 512px servidos corretamente
- [ ] `sw.js` na raiz do site com filtro de `same-origin`
- [ ] HTTPS ativo
- [ ] Lighthouse PWA Score maior ou igual a 90 (Chrome DevTools → Lighthouse)
- [ ] Testado no Chrome Android e Safari iOS

---

## ✨ A Filosofia Vincent AI Studios

> *"A barreira entre 'Site' e 'Aplicativo' deve sumir."*

Nosso padrão PWA-First:
- **Zero CDN externo** — todos os assets servidos localmente (sem Tracking Prevention)
- **Cache inteligente** — filtragem `same-origin` evita quebras com APIs externas
- **Auto-update silencioso** — `skipWaiting + controllerchange` sem intervenção do usuário
- **Leveza extrema** — Service Worker com menos de 3KB

---

## 💎 Vincent AI Studios
*A inteligência que evolui com você.*

Este manual é uma contribuição livre para a comunidade. Compartilhe, use e construa experiências incríveis.

---
© 2026 Vincent AI Studios · Todos os direitos reservados · `vincent-vangogh.web.app`
