# 🎯 Portal Finanças — página cinematográfica de scroll

> Uma página única em que o **símbolo do Bradesco** (o alvo vermelho vazado) nasce do centro, ganha espessura real em 3D, é atravessado por dentro como um portal e desemboca num corredor onde as aplicações do Portal Finanças passam uma a uma. Tudo dirigido pelo scroll, sem build.

![Preview](preview.gif) • Single-file • Sem build • 60fps • `pt-BR`

> **Preview:** `preview.gif` (grave rolando a página) e `preview.png` (frame estático). Se o GIF não carregar, sirva o `index.html` (veja _Como rodar_).

<p align="center">
  <img src="preview.svg" width="720" alt="Preview estático do símbolo 3D" />
  <br/>
  <em>Frame estático — no scroll ele gira, é atravessado e vira o corredor de aplicações</em>
</p>

---

## ✨ O que é

Demonstração de página inteira (referência de linguagem: [igloo.inc](https://www.igloo.inc/)) com o símbolo do Bradesco em **3D extrudado** como cenário fixo. Ao rolar, a narrativa passa por:

1. **Hero** — "Sobre o Portal Finanças", com o símbolo pequeno ao fundo.
2. **A marca** — o símbolo cresce e fica **interativo**: passar o mouse mostra o nome de cada pilar, arrastar gira em 3D.
3. **Cinco pilares** — Precisão, Tecnologia, Eficiência, Qualidade, Inovação. Cada seção acende uma parte do símbolo.
4. **Consolidação / travessia do portal** — o texto fica preso (sticky) e o resto do scroll é a câmera **mergulhando no símbolo**: as 5 peças centrais somem, o anel externo cresce e a câmera passa por dentro dele.
5. **Corredor das aplicações** — do outro lado do portal, um túnel onde as **6 telas do Portal Finanças** (`telas/*.svg`) passam pelas paredes e viram de frente ao entrar em foco. Clicar numa tela abre o detalhe; no detalhe, as **setas** (← → ou os botões) navegam entre as aplicações sem fechar.
6. **Encerramento** — a assinatura `#UnidosEvoluímos` e um botão _Voltar ao início_ que fecha e reabre uma íris no lugar de rolar 40 viewports.

A partir do início da travessia o símbolo deixa de ser interativo e vira só cenário (travessia, corredor, encerramento).

---

## 🧠 Princípio: tudo é função do scroll

O ponto central da engine: **nenhum visual da cena é animado no tempo**. Um único `requestAnimationFrame` (`frame()`) roda a cada quadro e recalcula câmera, símbolo, anel, túnel, telas e HUD como **função pura da posição de scroll**. "Mais lento" nunca é mudar uma curva de easing — é dar mais scroll para o mesmo trecho.

- A posição na narrativa (`segF`) é medida a partir do `offsetTop` real de cada seção (cache em `LM`, revalidado por `ResizeObserver`), não assumindo alturas iguais.
- `anime.js` é usado só para o que **não** é dirigido por scroll: reveal de texto por palavra, contador do loader, mola ao soltar o arraste do símbolo, íris do retorno ao topo.
- Scroll com inércia via `Lenis`; o `frame()` chama `lenis.raf` — sem loop próprio da lib.

---

## 🛠️ Stack

Tudo via `importmap` + CDN (jsdelivr). Sem `npm`, sem `vite`.

| Camada | Lib | Uso |
|---|---|---|
| **3D** | `three@0.160` | `ExtrudeGeometry` (`depth` + bisel) para dar espessura real ao símbolo; sombras `PCFSoft` |
| **Iluminação** | `three/addons` `RoomEnvironment` + `PMREMGenerator` | ambiente PBR sem carregar HDR; `ACESFilmicToneMapping` |
| **SVG → sólidos** | `three/addons` `SVGLoader` | lê o `path` exato do símbolo; cada uma das 7 ilhas vira um `Shape` extrudado independente (sem `evenodd` quebrado) |
| **Animação** | `animejs@4` | `animate`, `stagger`, `createTimeline`, `createSpring`, `svg`, `utils` |
| **Scroll** | `lenis@1` | inércia; o progresso dirige a cena no mesmo `rAF` |
| **Estilo** | CSS puro | fundo, grão SVG, vinheta, glow `radial-gradient`, tema dark→light por scroll |

> Não há mais postprocessing/bloom — o glow do símbolo vem da iluminação PBR + CSS.

---

## 📁 Estrutura

```
prototipo_icone_dinamico/
├─ index.html          # tudo: HTML + CSS + um <script type="module">
├─ telas/              # 6 mockups SVG das aplicações
│  ├─ cadastro.svg           painel-indicadores.svg   relatorios.svg
│  └─ aprovacoes.svg         conciliacao.svg          consultas.svg
├─ preview.gif / preview.png / preview.svg
├─ graphify-out/       # knowledge graph do projeto (navegação/consulta)
└─ README.md
```

O `index.html` é **single-file**. Blocos principais no markup:

- `<main>` com as seções da narrativa (`#hero`, `A marca`, 5 `[data-pillar]`, `.hold-sec`, `.apps-sec`, `.outro`).
- `<canvas>` fixo com a cena Three.
- `.corr-stage` + `.tunnel` — o corredor das aplicações, **fora do `<main>`** para poder aparecer ainda durante a travessia.
- `.rail` — navegação lateral por capítulos.
- `.loader`, `.rewind` (íris), `.zoom-bg` / `.zoom-cap` / `.zoom-nav` (detalhe da aplicação).

`data-sx` em cada seção é o deslocamento horizontal (parallax) do símbolo naquele trecho; `data-dark` marca as seções de fundo escuro para o tema.

---

## 🚀 Como rodar

O `importmap` e o `fetch` dos SVG das telas precisam de `http://` — abrir o arquivo direto (`file://`) não funciona.

```bash
npx serve .
# ou
python -m http.server 4599
```

Depois abra `http://localhost:<porta>/index.html` e role devagar de cima a baixo.

---

## 🎨 Como funciona (código)

### 1. Símbolo exato → sólidos

```js
const subPaths = new SVGLoader().parse(svgString).paths[0].subPaths;   // 7 ilhas
subPaths.forEach(sp => {
  const shape = new THREE.Shape(sp.getPoints(32));
  const geo = new THREE.ExtrudeGeometry(shape, {
    depth, bevelEnabled: true, bevelThickness: 1.2, bevelSize: 0.9, bevelSegments: 3,
  });
  geo.translate(-250, -250, -depth / 2);          // centro do viewBox 500
  const mesh = new THREE.Mesh(geo, [matFrente, matLado]);
  mesh.scale.y *= -1;                             // corrige o Y invertido do SVG
  alvo.add(mesh);
});
```

Cada ilha vira um sólido próprio — o anel externo (índices 0–1) é o que cresce e é atravessado na travessia; os 5 centrais somem cedo.

### 2. Travessia do portal

```js
// portalP = portalIn * (1 - pOut)  → entra e sai pela mesma função, ao contrário
const pDolly = Math.pow(portalP, 1.25);          // aproximação da câmera
camera.position.z = lerp(8.7 - s * 1.7, 2.37, pDolly);

// 5 peças centrais somem nos primeiros 12% do portal
const innerOp = 1 - smoothstep(clamp(portalP / 0.12, 0, 1));

// anel externo cresce e a câmera passa por dentro; some no cruzamento
anel.scale.multiplyScalar(1 + pDolly * pDolly * 3);
anelOp *= 1 - smoothstep(clamp((portalP - 0.79) / 0.10, 0, 1));

// o plano de sombra sai de cena nos primeiros 20%, senão o anel o atravessa
ground.visible = ground.material.opacity > 0.001;
```

### 3. Corredor com comprimento dinâmico

A `.apps-sec` é só um espaçador — é o comprimento dela que dá o scroll do corredor. Em vez de um valor fixo, ela é derivada do número de telas para o **ritmo por tela ficar constante** (≈ `VH_PER_APP` viewports cada), qualquer que seja o número de aplicações:

```js
const VH_PER_APP = 91.7;
function sizeCorridor() {
  const n = document.querySelectorAll(".corr-stage .app").length;
  if (n < 2) return;
  const tailVh = (holdSec.offsetHeight / innerHeight) * 100 * (1 - (P_END - I_HOLD - CORR_LEAD));
  const vh = Math.max(60, VH_PER_APP * (n - 1 + 0.06) - tailVh);   // desconta o trecho que cai dentro da Consolidação
  appsSec.style.height = appsSec.style.minHeight = vh.toFixed(1) + "vh";
}
```

### 4. Detalhe da aplicação + navegação por setas

Clique na tela em foco → FLIP (o retângulo de origem no corredor vira a animação ao contrário). No detalhe, `zoomNav(±1)` troca de aplicação por cross-fade, sem fechar, circular:

```js
const zoomNav = (dir) => {
  const nx = APPS[(APPS.indexOf(zoomedCard) + dir + APPS.length) % APPS.length];
  nx.style.cssText = "";          // limpa o estado congelado do corredor (visibility:hidden etc.)
  clearStyleCache(nx);
  // ...entra centralizado por cima, o anterior volta pro corredor (que segue congelado)
};
```

Fechar em qualquer aplicação colapsa de volta para o mesmo ponto do corredor (`zoomHome`, fixado no primeiro clique).

### 5. Tema dark → light

```js
root.style.setProperty("--bg",  hexMix(0xffffff, 0x0b0e15, smoother(d)));
root.style.setProperty("--ink", /* ... */);   // texto e realce acompanham na mesma curva
```

### 6. Acessibilidade

```js
const REDUCE = matchMedia("(prefers-reduced-motion: reduce)").matches;
// REDUCE: sem loader, sem Lenis, sem travessia; as telas viram uma grade simples
```

`lang="pt-BR"`, foco visível, `aria-label` na rail e nos botões de navegação.

---

## 🎛️ Customização

| O que | Onde | Dica |
|---|---|---|
| **Adicionar/remover uma aplicação** | um bloco `.app` em `.corr-stage` com `data-title`, `data-desc` e `<img src="telas/xxx.svg">` | corredor, contador `NN — total`, HUD e navegação por setas se ajustam sozinhos |
| **Ritmo do corredor** | `VH_PER_APP` | maior = cada tela dura mais scroll |
| **Ponto da travessia** | `P_START` / `P_END` (fração da seção Consolidação) | onde a câmera entra e sai do símbolo |
| **Cor do símbolo** | `const RED = 0xcc092f` + materiais | |
| **Espessura** | `depth: 22` no `ExtrudeGeometry` | `32` mais grossa, `14` mais fina |
| **Duração das seções** | `.hold-sec` (travessia, `470vh`), `.outro` (encerramento, `400vh`) | em vh |
| **Fundo / glow** | `.glow`, `.vignette`, `.grain` no CSS | troque o `radial-gradient` vermelho |
| **Texto estático** | tire o `data-reveal` do elemento | |

---

## ⚡ Performance

- `setStyle(el, prop, val)` com cache `WeakMap` — não reescreve estilo que já está aplicado (quem limpa `style` por fora chama `clearStyleCache`).
- `visibility: hidden` em vez de só `opacity: 0` para tirar camadas do compositor; o `<canvas>` sai do compositor quando 100% invisível.
- `renderer.shadowMap.autoUpdate` desligado quando o símbolo não está em cena.
- `antialias` e `devicePixelRatio` reduzidos em telas pequenas.
- Tudo num `rAF` só — sem `IntersectionObserver` dirigindo cena, sem mola brigando com o scroll.

---

## 📄 Licença

Protótipo interno. O símbolo e as marcas são do Bradesco.
