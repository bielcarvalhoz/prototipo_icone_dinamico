# 🎯 Protótipo — Ícone Dinâmico 3D no Scroll

> Um alvo vermelho que nasce do centro, ganha espessura real e gira em 3D conforme você rola a página. Feito com o seu SVG exato, sem aproximações.

![Preview](preview.gif) • Single-file • Sem build • 60fps

> **Preview:** `preview.gif` (grave rolando a página) e `preview.png` (frame estático). Se o GIF não carregar, abra o `index.html`.

<p align="center">
  <img src="preview.svg" width="720" alt="Preview estático do alvo 3D" />
  <br/>
  <em>Frame estático — no scroll ele gira e ganha espessura</em>
</p>

<details>
<summary>📹 Como gerar o <code>preview.gif</code></summary>

1. Abra o `index.html` em tela cheia (`F11`)
2. Grave rolando devagar de cima a baixo com:
   - **Windows:** [ScreenToGif](https://www.screentogif.com/) ou [LICEcap](https://www.cockos.com/licecap/)
   - **Mac:** `Cmd + Shift + 5` → Gravar
   - **CLI:** `ffmpeg -f gdigrab -framerate 30 -i desktop -qscale 0 preview.gif`
3. Salve como `preview.gif` na raiz — o README já referencia

> Dica: 800×450, 15fps, 4s já fica leve (&lt; 2MB). O `preview.svg` acima é só o frame estático.
</details>

---

## ✨ O que é

Página de demonstração com um **ícone/alvo em 3D extrudado** como plano de fundo fixo. Ao rolar:

1. **Cresce do centro** (`scale 0.002 → 0.0064` com `back.out`)
2. **Gira em 3D** `rotateY -42° → +84°` + `rotateX` sutil, com `perspective` real
3. **Barra grossa** visível na lateral — não é truque CSS, é `ExtrudeGeometry` com `depth 22` + bisel

O resto da página são seções em vidro (`backdrop-filter: blur`) por cima — copie o `index.html` e use como fundo do seu site.

---

## 🧠 De flor a alvo

O protótipo nasceu como uma **flor em SVG** montada no scroll e evoluiu para o seu **alvo** (`#CC092F`, vazado). Cada etapa foi iterada:

- **SVG puro + `stroke-dashoffset` + `scroll-timeline`** → leve, mas 2.5D
- **CSS `preserve-3d` + `translateZ`** → profundidade fake
- **Three.js + GSAP ScrollTrigger** → 3D real com luz, sombra e `scrub: true`

O alvo final mantém o **formato exato do seu SVG** — cada uma das 7 ilhas vermelhas vira um sólido separado extrudado, sem `evenodd` quebrado.

---

## 🛠️ Stack moderna

| Camada | Lib | Por que |
|---|---|---|
| **3D** | `three@0.160` via `importmap` | `ExtrudeGeometry` com `depth` + `bevel`, `DoubleSide`, sombras `PCFSoft` |
| **Scroll** | `gsap@3.12` + `ScrollTrigger` | `timeline({ scrub: 1.05 })` amarra progresso ao `scrollY`, `anticipatePin`, `back.out` |
| **SVG** | `three/addons/loaders/SVGLoader` | Lê seu `path` exato, cada `subPath` vira `Shape` sólido |
| **Estilo** | CSS puro | `perspective`, `backdrop-filter`, `radial-gradient` |

Sem `npm`, sem `vite`, sem `Three.js` pesado — tudo via CDN, abre com duplo clique.

---

## 📁 Estrutura

```
prototipo_icone_dinamico/
├─ index.html   # tudo aqui: HTML + CSS + JS (Three + GSAP)
├─ README.md
└─ .codegraph/  # índice do projeto
```

O `index.html` é **single-file**: `<canvas id="c">` fixo atrás + `.wrap` com 3 seções de `110vh` pra dar scroll.

---

## 🚀 Como usar

```bash
# 1. Abrir direto
# duplo clique em index.html

# 2. Servir local (evita CORS do importmap em alguns browsers)
npx serve .
# http://localhost:3000
```

Role devagar de `top` a `bottom` e veja a montagem. `Ctrl+F5` após editar.

---

## 🎨 Como funciona (código)

### 1. SVG exato → sólidos

```js
const data = new SVGLoader().parse(svgString); // seu path com 7 ilhas
data.paths[0].subPaths.forEach(sp => {
  const shape = new THREE.Shape(sp.getPoints(32));
  const geo = new THREE.ExtrudeGeometry(shape, {
    depth: 22, bevelEnabled: true, bevelThickness: 1.2, bevelSize: 0.9
  });
  geo.translate(-250, -250, -depth/2); // centro do viewBox 500
  const mesh = new THREE.Mesh(geo, [matFront, matSide]);
  mesh.scale.y *= -1; // corrige Y invertido do SVG
  alvo.add(mesh);
});
```

Cada ilha é um sólido independente — nada de `evenodd` vazado, nada de `center()` por ilha que desalinha.

### 2. Espessura

```js
const matFront = new THREE.MeshStandardMaterial({ color: 0xCC092F, side: DoubleSide });
const matSide  = new THREE.MeshStandardMaterial({ color: 0x7a051c, side: DoubleSide });
// depth: 22 → barra grossa visível quando gira
```

`DoubleSide` garante frente e verso preenchidos.

### 3. Scroll

```js
gsap.set(alvo.scale, { x: 0.0022, y: 0.0022, z: 0.0022 });
gsap.set(alvo.rotation, { y: -0.72, x: 0.15 });

gsap.timeline({
  scrollTrigger: { trigger: document.body, start: "top top", end: "bottom bottom", scrub: 1.05 }
})
.to(alvo.scale, { x: 0.0064, y: 0.0064, z: 0.0064, duration: 0.72, ease: "back.out(1.15)" }, 0)
.to(alvo.rotation, { y: 0.85, x: -0.11, duration: 0.72, ease: "none" }, 0)
.to(alvo.rotation, { y: 1.45, x: -0.15, duration: 0.28, ease: "none" }, 0.72);
```

### 4. Acessibilidade

```js
if (matchMedia('(prefers-reduced-motion: reduce)').matches) {
  gsap.set(alvo, { scale: 0.0064, rotation: { x:0, y:0 } });
}
```

Sem animação se o usuário prefere.

---

## 🎛️ Customização

| O que | Onde | Dica |
|---|---|---|
| **Cor** | `matFront.color` / `matSide.color` | Troque `0xCC092F` |
| **Espessura** | `depth = 22` | `32` → mais grossa, `14` → mais fina |
| **Zoom** | `camera.position.set(0,0.6,7.8)` + `alvo.scale 0.0064` | Diminua `z` ou aumente `scale` pra mais perto |
| **Giro** | `rotation.y -0.72 → 1.45` | `Math.PI*2` pra 360° |
| **Fundo** | `.bg` `radial-gradient` | Troque o degradê atrás do vazado |

Troque o `svgString` por qualquer SVG — o `subPaths` já cuida.

---

## ⚡ Performance

- `renderer.setPixelRatio(min(devicePixelRatio, 2))`
- `shadowMap: 2048²` só no `sun`, `PCFSoft`
- `antialias: true` + `ACESFilmicToneMapping`
- `alpha: true` no canvas deixa o degradê aparecer sem custo

---

## ♿ A11y

- Vazado mostra o fundo — contraste mantido
- `prefers-reduced-motion` → estático
- Sem `overflow: hidden` que quebra `scroll-timeline`

---

## 📄 Licença

Protótipo — use como quiser. SVG original é seu.

Feito com ☕ e muito `Ctrl+F5` pra corrigir o vazado.
