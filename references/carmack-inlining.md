# Carmack on Inlining — Email Original (2007) + Comentário (2014)

> Email enviado por John Carmack à equipe do id Software em 13 de março de 2007.
> Comentário adicionado por ele mesmo em setembro de 2014.
> Preservado aqui para agentes de referência.

---

## Resumo Executivo

Carmack argumenta que funções chamadas de um único lugar devem ser inlined (escritas diretamente no corpo da função chamadora). Isso:

1. **Aumenta consciência** — você vê todo o código que executa sequencialmente
2. **Elimina bugs de "called in wrong state"** — a função não pode ser chamada do lugar errado porque não existe como entidade separada
3. **Torna latência explícita** — em game loops, ordem de execução importa
4. **Reduz surface area** — menos funções = menos lugares para bugs

---

## As Três Formas (Style A, B, C)

```c
// Style A — histórico do id (Carmack usava isso)
void MinorFunction1(void) { }
void MinorFunction2(void) { }
void MinorFunction3(void) { }

void MajorFunction(void) {
    MinorFunction1();
    MinorFunction2();
    MinorFunction3();
}

// Style B — preferência de alguns (ordem invertida)
void MajorFunction(void) {
    MinorFunction1();
    MinorFunction2();
    MinorFunction3();
}
void MinorFunction1(void) { }
// ...

// Style C — Michael Abrash style, Carmack passou a preferir
void MajorFunction(void) {
    // MinorFunction1
    { /* código inline aqui */ }

    // MinorFunction2
    { /* código inline aqui */ }

    // MinorFunction3
    { /* código inline aqui */ }
}
```

**Carmack converteu o código de Abrash de Style C para Style A** quando eram jovens. Anos depois, reconheceu que Abrash estava certo.

---

## As Regras Destiladas

Do email original:

1. **Se uma função é chamada de um único lugar → considere inline**
2. **Se é chamada de múltiplos lugares → veja se pode consolidar com flags, depois inline**
3. **Se há múltiplas versões de uma função → faça uma com mais parâmetros (defaulted)**
4. **Se é quase-pura (poucas referências a estado global) → empurre para pureza total**
5. **Use `const` em parâmetros e funções onde a função deve ser usada em múltiplos lugares**
6. **Minimize complexidade de control flow — prefira execução consistente a "otimizar" trabalho desnecessário**

---

## O Argumento do Latency Frame

Um dos argumentos mais práticos do email:

> *"It is very easy for frames of operational latency to creep in when operations are done deeply nested in various subsystems, and things evolve over time. [...] If everything is just run out in a 2000 line function, it is obvious which part happens first."*

Em game development (e sistemas real-time em geral), a **ordem** importa tanto quanto o **quê**. Quando o código está inlined numa função longa, a ordem é óbvia. Quando está espalhado em chamadas profundas, latência de frames aparece de forma sorrateira.

Carmack quase enviou o DOOM 3 BFG Edition com um frame de latência de input por exatamente esse motivo — e só descobriu por causa desta análise.

---

## O Comentário de 2014 — Evolução para Funcional

```
"In the years since I wrote this, I have gotten much more bullish about 
pure functional programming, even in C/C++ where reasonable.

The real enemy addressed by inlining is unexpected dependency and mutation 
of state, which functional programming solves more directly and completely.

However, if you are going to make a lot of state changes, having them all 
happen inline does have advantages; you should be made constantly aware of 
the full horror of what you are doing. When it gets to be too much to take, 
figure out how to factor blocks out into pure functions (and don't let them 
slide back into impurity!)."
```

**Tradução prática:**
- Se seu código muta muito estado → inline para ter consciência total do horror
- Quando o horror for demais → extraia para funções puras
- Nunca deixe funções puras se tornarem impuras novamente

---

## O Argumento dos Bugs de Copy-Paste

Carmack rastreou seus próprios bugs por um período e ficou surpreso:

```c
// Ele costumava escrever assim:
v[0] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*bytePitch+j*8+0));
v[1] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*bytePitch+j*8+1));
v[2] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*bytePitch+j*8+2));
v[3] = HF_MANTISSA(*(halfFloat_t *)((byte *)data + i*bytePitch+j*8+3));
```

*"I now strongly encourage explicit loops for everything, and hope the compiler unrolls it properly. A significant number of my bugs related to things like this."*

**Lesson:** Se você está mudando só um dígito/índice entre linhas, é um loop disfarçado. Transforme em loop.
