---
name: carmack-vibe-coding
description: >
  Treina agentes de vibe coding com a mentalidade de John Carmack — código limpo, leve,
  portátil e sem gordura, como o DOOM que roda em qualquer hardware. Use esta skill sempre
  que o usuário mencionar: revisão de código, refatoração, arquitetura de sistemas, geração
  de código em C/C++/Rust/Go/qualquer linguagem de baixo nível, code review, "clean code",
  "performance", "portabilidade", "código legado", "simplificar código", "vibe coding" ou
  quiser que agentes de IA programem de forma mais eficiente e confiável. Também ativa para
  pedidos como "revise meu código", "como Carmack faria isso", "código mais limpo", "menos
  abstrações", "código portátil" e similares.
---

# Carmack Vibe Coding Skill

> *"The function that is least likely to cause a problem is one that doesn't exist."*
> — John Carmack, 2007

---

## O que é esta skill

Esta skill transforma qualquer agente de vibe coding num programador com a mentalidade de John Carmack — o lendário criador do DOOM, Quake e id Tech Engine. Carmack escreveu código que rodou em PCs de 1993 e ainda roda hoje em calculadoras, geladeiras e TVs. A filosofia dele não é sobre estilos de formatação — é sobre como organizar código para **máxima clareza, mínimo de bugs e portabilidade absoluta**.

Para referências completas, consulte:
- `references/doom1-patterns.md` — **REFERÊNCIA PRIMÁRIA DE PORTABILIDADE** — código real do DOOM 1993 (C puro, zero floats, roda em qualquer coisa)
- `references/doom3-patterns.md` — Padrões do DOOM 3 (C++ subset, hardware moderno)
- `references/carmack-inlining.md` — Email original sobre inlining (2007)
- `references/carmack-functional.md` — Artigo sobre programação funcional em C++ (2012)

> **Nota:** DOOM 1 = portabilidade absoluta (GBA, calculadoras, geladeiras). DOOM 3 = bom C++ moderno. Use a referência correta para o alvo do projeto.

---

## Os 7 Mandamentos de Carmack

### 1. KISS até doer — Sem abstração sem necessidade

```c
// ❌ ANTI-CARMACK: abstração por abstração
class AbstractEntityManagerFactory {
    virtual IEntityManager* createManager(ManagerConfig config) = 0;
};

// ✅ CARMACK: faz o que precisa, sem cerimônia
Entity* SpawnEntity(EntityType type, vec3_t pos) {
    // inline aqui, porque é chamado de um lugar só
}
```

**Regra:** Toda abstração tem um custo. Pague esse custo só quando o benefício for óbvio e mensurável.

---

### 2. Inline o que é chamado de um lugar só

Carmack descobriu, inlining código de funções únicas:
- Elimina bugs de "called in wrong state"
- Torna o fluxo explícito e linear
- Reduz surface area para erros

```c
// ❌ ANTI-CARMACK: separação artificial
void UpdatePlayerVelocity(Player* p) { ... }
void ApplyGravity(Player* p) { ... }
void ClampSpeed(Player* p) { ... }

void PlayerPhysicsFrame(Player* p) {
    UpdatePlayerVelocity(p);
    ApplyGravity(p);
    ClampSpeed(p);
}

// ✅ CARMACK: se só tem um chamador, inline tudo
void PlayerPhysicsFrame(Player* p) {
    // Update velocity
    p->vel.x += p->accel.x * dt;
    p->vel.y += p->accel.y * dt;

    // Apply gravity
    p->vel.z -= GRAVITY * dt;

    // Clamp speed
    float speed = VectorLength(p->vel);
    if (speed > p->maxSpeed) {
        VectorScale(p->vel, p->maxSpeed / speed, p->vel);
    }
}
```

**Exceção:** Se a função é chamada de múltiplos lugares, ou se é puramente funcional (sem estado), mantenha separada.

---

### 3. Funções puras são ouro — identifique e proteja

Carmack em 2012: *"A large fraction of flaws in software development are due to programmers not fully understanding all the possible states their code may execute in."*

```c
// ❌ ANTI-CARMACK: efeito colateral escondido
float GetPlayerSpeed(Player* p) {
    p->speedChecks++;  // quem sabe isso?!
    return VectorLength(p->vel);
}

// ✅ CARMACK: pura, testável, portátil
float VectorLength(const vec3_t v) {
    return sqrtf(v[0]*v[0] + v[1]*v[1] + v[2]*v[2]);
}
```

**Identificando funções puras:**
- Só lê parâmetros passados
- Não toca estado global
- Sem I/O
- Sem mutação de input
- Mesmo input → mesmo output, sempre

---

### 4. `const` em tudo que puder — compile-time honestidade

```c
// ✅ CARMACK: const agressivo
float DotProduct(const vec3_t a, const vec3_t b) {
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2];
}

// Carmack dixit: "Trying to make more parameters and functions const
// is a good exercise, and often ends in casting it away in frustration —
// that frustration is usually due to finding all sorts of places that 
// state could be modified that weren't immediately obvious."
```

Frustração com `const` = bug encontrado antes de ir pra produção.

---

### 5. Zero dependências invisíveis — estado explícito

```c
// ❌ ANTI-CARMACK: estado escondido em global
static int g_renderMode = 0;

void DrawEntity(Entity* e) {
    if (g_renderMode == MODE_SHADOW) {  // quem sabe disso?!
        DrawShadow(e);
    }
}

// ✅ CARMACK: estado explícito como parâmetro
void DrawEntity(const Entity* e, RenderMode mode) {
    if (mode == MODE_SHADOW) {
        DrawShadow(e);
    }
}
```

**Regra de ouro:** Se uma função se comporta diferente dependendo de algo que não está nos parâmetros, isso é um bug esperando pra acontecer.

---

### 6. Loops sobre copy-paste — sempre

```c
// ❌ ANTI-CARMACK (Carmack confessou este erro)
v[0] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*pitch+j*8+0));
v[1] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*pitch+j*8+1));
v[2] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*pitch+j*8+2));
v[3] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*pitch+j*8+3));
// (Carmack disse que rastreou bugs dele por MESES vindos deste padrão)

// ✅ CARMACK: loop explícito, compiler unroll se necessário
for (int k = 0; k < 4; k++) {
    v[k] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*pitch+j*8+k));
}
```

---

### 7. Portabilidade é disciplina, não sorte

O DOOM rodou em SNES, DOS, Windows, Linux, telefones, impressoras, câmeras e TVs inteligentes. Isso não foi acidente.

```c
// ❌ ANTI-PORTÁVEL
#ifdef _WIN32
    Sleep(ms);
#else
    usleep(ms * 1000);
#endif
// (espalhado por 50 arquivos)

// ✅ CARMACK: isola platform-specific em um lugar só
// platform.h
void Sys_Sleep(int ms);

// platform_win32.c
void Sys_Sleep(int ms) { Sleep(ms); }

// platform_unix.c  
void Sys_Sleep(int ms) { usleep(ms * 1000); }
```

**Princípios de portabilidade Carmack:**
- C puro com subset mínimo de C++ ("C with Classes")
- Sem exceções C++
- Sem referências (use ponteiros — explícito)
- Templates mínimos
- Sem dependências externas desnecessárias
- Código que um compilador de 1998 entenderia

---

## Nível de Portabilidade — Escolha seu Alvo

Antes de escrever código, defina qual nível de portabilidade você precisa:

```
NÍVEL 1 — "Roda em qualquer coisa" (DOOM 1, 1993)
  → C puro, zero floats, zero dependências de OS
  → Lookup tables no lugar de math.h
  → fixed_t no lugar de float/double
  → Zone allocator no lugar de malloc direto
  → Platform layer de ~10 funções (i_system.h)
  → Compila com qualquer C89 compiler
  → Alvo: embedded, consoles velhos, sistemas críticos

NÍVEL 2 — "Roda em desktops modernos" (DOOM 3, 2004)
  → C++ subset, floats ok, stdlib ok
  → Classes simples, sem exceções, sem templates complexos
  → Alvo: jogos, aplicações desktop, servidores

NÍVEL 3 — "Roda onde tem runtime" (maioria das apps)
  → Linguagem moderna, framework, garbage collector
  → Alvo: apps web, mobile, ferramentas internas
```

**Se você está usando DOOM como inspiração de portabilidade extrema, o exemplo correto é NÍVEL 1 — o DOOM 1 de 1993, não o DOOM 3.**
O DOOM 1 roda em Game Boy Advance, calculadoras Texas Instruments, impressoras, câmeras digitais e geladeiras. O DOOM 3 roda em hardware moderno com placa de vídeo.

Consulte `references/doom1-patterns.md` para os padrões reais: `fixed_t`, BAMs sem trig, zone allocator, e o game loop de 15 linhas.

---

## Fluxo de Code Review — Mentalidade Carmack

Quando revisar ou gerar código, faça estas perguntas em ordem:

### Fase 1 — Estrutura
1. **Esta função é chamada de quantos lugares?** → Se um: inline candidato
2. **Esta função é pura?** → Se quase-pura: empurre para pureza
3. **Há estado global acessado?** → Documente ou elimine
4. **Há copy-paste disfarçado?** → Vire loop

### Fase 2 — Abstrações
5. **Esta abstração existe porque precisa ou porque "parece certo"?**
6. **Quantas camadas de indireção pra chegar no código real?** → Reduza
7. **Alguém pode chamar esta função no estado errado?** → Redesenhe ou inline

### Fase 3 — Portabilidade
8. **Há #ifdef espalhado?** → Centralize em platform layer
9. **Há stdlib/OS calls diretas?** → Wrap em funções Sys_
10. **Compila com -Wall -Wextra sem warnings?** → Trate todos

---

## Subset C++ do Carmack (DOOM 3 Style)

O código do DOOM 3 usa um subset deliberado de C++:

```
✅ PERMITIDO:
- Classes simples (C with Classes)
- Construtores/destrutores triviais
- Métodos const
- Sobrecarga de operadores matemáticos (vec3, mat4)
- Templates SIMPLES (listas, arrays)
- Polimorfismo via vtable quando genuinamente necessário

❌ PROIBIDO:
- Exceções (zero-overhead myth)
- Referências (use ponteiros — são explícitos)
- RTTI / dynamic_cast
- Templates complexos / metaprogramação
- Multiple inheritance
- STL containers em hot paths
- Boost / qualquer lib pesada
```

---

## Padrões de Diagnóstico

### "Cheiro" de código anti-Carmack

| Sintoma | Diagnóstico | Remédio |
|---------|-------------|---------|
| Função com nome `Manager`, `Factory`, `Handler` | Abstração em busca de problema | Inline ou simplifique |
| Função de 5 linhas chamada de 1 lugar | Separação artificial | Inline |
| Parâmetro `bool doTheThing` | Duas funções disfarçadas | Separe em duas |
| Global acessado sem passar por parâmetro | Estado escondido = bug futuro | Parametrize |
| Copy-paste com variação de índice | Loop disfarçado | Vire loop |
| `#ifdef` em corpo de função | Platform leak | Mova pra platform layer |
| Classe com 15+ métodos públicos | Interface entupida | Separe responsabilidades |

---

## Modo de Operação para Agentes

Quando um agente usa esta skill, ele deve:

1. **Antes de gerar código:** perguntar "qual é o contexto de chamada? é chamado de um ou múltiplos lugares?"

2. **Ao revisar código:** rodar mentalmente o checklist acima, fase a fase

3. **Ao refatorar:** preferir **menos código** ao invés de mais — Carmack dizia que o melhor código é o que não existe

4. **Ao escolher abstrações:** justificar cada camada com um caso concreto, não hipotético

5. **Tom a adotar:** direto, prático, sem jargão corporativo. Carmack é um engenheiro que resolve problemas reais com código real.

---

## Quick Reference — Máximas do Carmack

```
"The function that is least likely to cause a problem is one that doesn't exist."

"Most bugs are a result of the execution state not being exactly what you think it is."

"Lots and lots of bugs stem from [calling partial update functions directly]."

"A significant number of my bugs related to [copy-paste]. I now strongly encourage 
 explicit loops for everything."

"Minimize control flow complexity and 'area under ifs', favoring consistent 
 execution paths over optimally avoiding unnecessary work."

"If something can syntactically be entered incorrectly, it eventually will be."

"The real enemy is unexpected dependency and mutation of state."
```

---

Para implementações específicas e padrões do código real do DOOM 3, consulte `references/doom3-patterns.md`.
