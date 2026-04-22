# Padrões Reais do Código DOOM 3

> Fonte: id Software open-source release (DOOM 3 BFG Edition)
> Analisado por: Fabien Sanglard e comunidade

---

## O Subset C++ do id Software

O DOOM 3 é escrito em C++, mas usa um subset deliberadamente restrito que Fabien Sanglard descreveu como "C with Classes" — flui pelo cérebro com mínima resistência.

### Sem exceções

```cpp
// DOOM 3 não usa try/catch em nenhum lugar do engine
// Error handling é feito com códigos de retorno e asserts
void idLib::Error(const char* fmt, ...) {
    // loga o erro, chama handler, termina limpo
    // sem stack unwinding, sem surpresas
}
```

### Ponteiros, não referências

```cpp
// ✅ DOOM 3 style — explícito
void idRenderWorld::AddEntityDef(const renderEntityParms_t* re);

// ❌ NÃO é o estilo deles
void AddEntityDef(const renderEntityParms_t& re);
```

Ponteiros são honestos: você sabe que é um ponteiro. Referências escondem indireção.

### Math types simples e funcionais

```cpp
// idVec3 — operações retornam novo valor (funcional)
idVec3 idVec3::Normalized() const {
    float len = Length();
    return idVec3(x/len, y/len, z/len);
}

// ❌ versão antiga (self-mutating) que Carmack citou como erro
void idVec3::Normalize() {
    float len = Length();
    x /= len; y /= len; z /= len;
}
```

---

## O Game Loop — Linear e Explícito

O coração do DOOM é um loop de jogo simples e plano:

```c
// Estilo DOOM original — tudo explícito, sequencial
void D_DoomLoop(void) {
    while (1) {
        // 1. Processar input — SEMPRE, sem condicionais
        I_StartTic();
        
        // 2. Rodar lógica de jogo
        G_Ticker();
        
        // 3. Renderizar
        D_Display();
        
        // Execute tudo, iniba resultados se necessário
        // NÃO skip frames baseado em estado
    }
}
```

**Princípio:** Execute sempre, depois descarte se necessário. Não tente ser "esperto" e pular execução — isso gera bugs de estado inconsistente.

---

## Nomenclatura Carmack

O código id usa prefixos sistemáticos que indicam escopo e subsistema:

```c
// Prefixos de subsistema:
// R_   = Renderer
// G_   = Game logic  
// P_   = Player
// M_   = Menu
// I_   = Interface (platform layer)
// S_   = Sound
// W_   = WAD file system
// Z_   = Zone memory manager

// Exemplos reais:
R_RenderScene();
G_Ticker();
P_PlayerThink();
I_InitGraphics();
S_StartSound();

// Platform layer isolado:
I_GetTime();    // abstrai clock do OS
I_Error();      // abstrai saída de erro
I_Quit();       // abstrai shutdown do OS
```

Esta convenção torna imediatamente óbvio de qual subsistema uma função pertence — sem precisar abrir o header.

---

## Zone Memory Allocator — Portabilidade por Design

O DOOM tem seu próprio gerenciador de memória (`z_zone.c`) — uma decisão clássica de Carmack para portabilidade total:

```c
// Wrapper sobre malloc/free do sistema
// Toda alocação passa por aqui — zero malloc() direto no código de jogo
void* Z_Malloc(int size, int tag, void* user);
void  Z_Free(void* ptr);
void  Z_FreeTags(int lowtag, int hightag);

// Benefícios:
// 1. Debug de memória centralizado
// 2. Portável para sistemas sem malloc (consoles, embedded)
// 3. Profiling e tracking num lugar só
// 4. Nunca vaza memória entre níveis (Z_FreeTags(PU_LEVEL, PU_LEVEL))
```

**Lição:** Envolva dependências de sistema em wrappers desde o início.

---

## O fast inverse square root — Pragmatismo sobre Pureza

```c
// O código mais famoso do Quake — atribuído a Carmack/id
float Q_rsqrt(float number) {
    long i;
    float x2, y;
    const float threehalfs = 1.5F;

    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;           // evil floating point bit level hacking
    i  = 0x5f3759df - ( i >> 1 );   // what the f**k?
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) ); // 1st iteration
//  y  = y * ( threehalfs - ( x2 * y * y ) ); // 2nd iteration (remover)

    return y;
}
```

**Comentário do próprio código:** `// what the f**k?`

**Lição de Carmack aqui:** Quando performance importa de verdade, pragmatismo supera elegância. Mas documente o hack — e isole ele. Esta função é pura, portátil, e o "hack feio" está contido em 4 linhas.

---

## Padrão de Configuração — Structs, não Classes Complexas

```cpp
// DOOM 3 usa structs de parâmetros — sem getters/setters
typedef struct {
    idVec3          origin;
    idMat3          axis;
    qhandle_t       hModel;
    int             entityNum;
    idVec3          bounds[2];
    // ...
} renderEntityParms_t;

// Passagem direta, sem encapsulamento artificial
void R_AddEntitySurfaces(renderEntityParms_t* entity);
```

**Por que isso é melhor:**
- Sem getter/setter boilerplate
- Debuggável diretamente no watch window
- Copiável/serializável trivialmente
- Legível sem IDE

---

## Anti-padrões que Carmack ativamente evitou

### 1. O PartialUpdate Trap

```cpp
// ❌ Perigoso — convida a ser chamado "otimizando" pra pular FullUpdate
void FullUpdate()   { PartialUpdateA(); PartialUpdateB(); }
void PartialUpdateA() { ... }
void PartialUpdateB() { ... }

// Seis meses depois, alguém faz:
PartialUpdateB(); // "eu só preciso de B!"
// = bug garantido quando A tem estado que B depende

// ✅ Carmack: use flag para inibir, não para pular
void FullUpdate(bool skipA = false) {
    if (!skipA) { /* update A inline */ }
    /* update B inline */
}
```

### 2. O "Clever" Optimization Early

```c
// ❌ Anti-Carmack: otimizar antes de medir
void GameLoop() {
    if (entityNeedsUpdate) {  // "otimização" que gera bugs de estado
        UpdateEntity();
    }
}

// ✅ Carmack: execute sempre, mida depois
void GameLoop() {
    UpdateEntity();  // sempre
    // se for lento, profile e prove antes de otimizar
}
```

### 3. O Estado Espalhado

```c
// ❌ Estado espalhado em múltiplos globals
extern int   g_playerHealth;
extern float g_playerSpeed;  
extern bool  g_playerInvincible;
// (25 arquivos diferentes acessam esses globals)

// ✅ Estado agrupado, ownership claro
typedef struct {
    int   health;
    float speed;
    bool  invincible;
    // tudo que pertence ao player
} player_t;

extern player_t g_player; // um ponto de acesso
```
