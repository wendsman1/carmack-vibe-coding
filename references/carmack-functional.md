# Carmack on Functional Programming in C++ (2012)

> Artigo escrito por John Carmack em 30 de abril de 2012.
> Preservado aqui para referência de agentes.

---

## A Tese Central

> *"A large fraction of the flaws in software development are due to programmers not fully understanding all the possible states their code may execute in."*

Programação funcional resolve isso tornando o estado **explícito**. Não requer Haskell ou Lisp — você pode aplicar princípios funcionais em qualquer linguagem, incluindo C.

---

## O que é uma Função Pura

```c
// Função PURA — todos esses critérios:
// 1. Só lê parâmetros passados
// 2. Não toca estado global
// 3. Sem I/O
// 4. Não muta inputs
// 5. Mesmo input → mesmo output, sempre

float DotProduct(const vec3_t a, const vec3_t b) {
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2];
}

// Função IMPURA — exemplos do que contamina:
float GetSpeed(Player* p) {
    p->speedChecks++;           // ← muta estado externo
    LogAccess("speed");         // ← I/O
    return speedMultiplier * VectorLength(p->vel);  // ← global
}
```

---

## Benefícios Práticos (não teóricos)

### Thread Safety
Funções puras com parâmetros por valor são **completamente thread-safe** sem locks. Você pode paralelizar trivialmente.

### Reusabilidade/Portabilidade
```c
// Função pura é trivialmente portável
// Não tem snowball effect de dependências
float Lerp(float a, float b, float t) {
    return a + t * (b - a);
}
// → Copie para qualquer projeto. Funciona. Sem dependências.
```

### Testabilidade
```c
// Funções puras são triviais de testar
assert(Lerp(0.0f, 10.0f, 0.5f) == 5.0f);
assert(DotProduct(v1, v2) == expected);
// Sem setup, sem mocks, sem harness complexo
```

### Manutenibilidade
> *"The bounding of both input and output makes pure functions easier to re-learn when needed, and there are less places for undocumented requirements regarding external state to hide."*

---

## O Contínuo de Pureza

Carmack é pragmático — não é tudo ou nada:

```
Spaghetti state  →  Mostly-pure  →  Almost-pure  →  Completely pure
     ↑                                                      ↑
  pior caso                                           ideal teórico
  
O salto de "spaghetti" para "mostly-pure" vale MAIS do que de "almost-pure" para "completely-pure".
Foco no que move a agulha mais.
```

Uma função que toca um global counter mas não tem outros efeitos colaterais ainda recolhe **a maioria dos benefícios** de uma função pura.

---

## OOP vs Funcional — A Tensão

```
"OO makes code understandable by encapsulating moving parts." 
                                              — Michael Feathers

"FP makes code understandable by minimizing moving parts."
```

Carmack não é anti-OOP, mas identifica o problema central:

```cpp
// Métodos de objeto que mutam estado são anti-funcionais
player->SetHealth(100);   // ← muta objeto
player->TakeDamage(30);   // ← muta objeto
// Sequência de chamadas importa — estado escondido

// Estilo mais funcional:
PlayerState newState = ApplyDamage(currentState, 30);
// Estado explícito, transformação pura, testável
```

---

## Implicações de Performance — Honestidade

Carmack é honesto: programação funcional às vezes tem custo:

```cpp
// ❌ Funcional DEMAIS — não faça isso em hot path
Framebuffer DrawTriangle(Framebuffer fb, Triangle tri) {
    Framebuffer newFb = fb;  // copia framebuffer inteiro!
    // ... desenha no newFb
    return newFb;
}

// ✅ Pragmático — aceita ponteiro de output mas deixa input const
void DrawTriangle(const Triangle* tri, Framebuffer* fb) {
    // Não é "puro" mas é o certo aqui
}
```

> *"In almost all cases, directly mutating blocks of memory is the speed-of-light optimal case, and avoiding this is spending some performance. Most of the time this is of only theoretical interest."*

Aplique pureza onde ela não tem custo de performance. Onde tem custo real e medido, seja pragmático.

---

## Action Items do Carmack

1. **Faça um survey de funções não-triviais no seu codebase** — rastreie todo estado externo que elas podem acessar e todas as modificações possíveis. Isso vira documentação valiosa.

2. **Na próxima feature nova** — pense em termos de: qual é o input real? qual é a computação pura? qual é o output? Tente estruturar assim.

3. **Ao debugar** — note conscientemente quando estado mutável e parâmetros escondidos estão obscurecendo o que acontece.

4. **Nos seus utility objects** — modifique métodos para retornar cópias em vez de auto-mutar. Jogue `const` na frente de quase todas as variáveis que não são iteradores.

---

## Regra Prática para Agentes

Ao analisar código, classifique cada função em:

| Tipo | Descrição | Ação |
|------|-----------|------|
| **Pura** | Só parâmetros, sem efeitos colaterais | Marque como pura, não toque |
| **Quase-pura** | Um ou dois globais, sem mutação de input | Empurre para pureza |
| **Impura controlada** | Muta estado, mas de forma isolada e óbvia | Documente efeitos |
| **Impura tóxica** | Web de dependências e estado escondido | Refatore urgente |

Foque na última categoria — é onde vivem os bugs de produção.
