# Padrões do DOOM 1 (1993) — A Portabilidade Real

> Baseado no código-fonte real do DOOM 1.10, liberado por Carmack em 1997.
> Repositório oficial: github.com/id-Software/DOOM
>
> **Este é o código que roda em Game Boy Advance, calculadoras, impressoras,
> babás eletrônicas, câmeras, osciloscópios e ATMs.**
> O DOOM 3 roda em hardware moderno. O DOOM 1 roda em qualquer coisa.

---

## Por que o DOOM 1 é mais radical que o DOOM 3

| Aspecto | DOOM 1 (1993) | DOOM 3 (2004) |
|---------|--------------|--------------|
| Linguagem | C puro | C++ subset |
| Hardware alvo | 4MB RAM, 486, **sem FPU** | OpenGL, hardware 3D |
| Float | **Zero** — proibido por constraint | Uso livre |
| Abstração | Structs + funções globais | Classes com herança |
| Portabilidade | Roda em qualquer CPU com C compiler | Hardware moderno |
| Linhas de código | ~30k linhas | ~600k linhas |

O DOOM 3 é "bom código C++". O DOOM 1 é **sobrevivência com disciplina extrema**.

---

## A Decisão Mais Radical: Zero Floats

Em 1993, só as máquinas high-end tinham FPU. A decisão de Carmack: **banir floats do engine inteiro**.

```c
// doomdef.h — a fundação de tudo
// Fixed point, 16-bit integer, 16-bit fraction
typedef int fixed_t;

#define FRACBITS    16
#define FRACUNIT    (1<<FRACBITS)   // = 65536, representa "1.0"

// Operações com fixed_t
#define FixedMul(a,b)   ((fixed_t)(((long long)(a)*(b))>>FRACBITS))
#define FixedDiv(a,b)   (((abs(a)>>14) >= abs(b)) ? ((a^b)<0 ? MININT : MAXINT) : FixedDiv2(a,b))
```

**O que isso significa na prática:**
```c
// "1.5" em fixed_t = 1.5 * 65536 = 98304
// "3.14159" em fixed_t = 3.14159 * 65536 = 205887

fixed_t playerX = 128 * FRACUNIT;  // posição X do player = 128.0
fixed_t speed   = FRACUNIT / 2;    // velocidade = 0.5 unidades/tick

// Multiplicar: (128.0 * 0.5) = 64.0
fixed_t result = FixedMul(playerX, speed);  // = 64 * FRACUNIT
```

**Por que isso é mais portável:**
- Funciona em qualquer CPU de 32 bits
- Resultado 100% determinístico em toda plataforma
- Sem floating-point rounding differences entre OSes
- Demos do DOOM são reproduzíveis bit-a-bit por isso

---

## Binary Angular Measurement (BAMs) — Sem trigonometria em runtime

Sem FPU, Carmack não pode chamar `sin()` e `cos()` em runtime. Solução: **lookup tables pré-computadas** e um sistema de ângulos que é só inteiros.

```c
// doomdef.h
#define ANGLETOFINESHIFT    19
#define FINEANGLES          8192    // tabela de 8192 entradas
#define FINEMASK            (FINEANGLES-1)

// angle_t é um inteiro de 32 bits — 0 a 0xFFFFFFFF = círculo completo
typedef unsigned angle_t;

#define ANG45   0x20000000  // 45 graus como inteiro
#define ANG90   0x40000000  // 90 graus
#define ANG180  0x80000000  // 180 graus
#define ANG270  0xc0000000  // 270 graus

// Tabelas de lookup (computadas em tempo de compilação/startup)
// finesine[8192]   — sin para todos os ângulos
// finecosine[8192] — cos (apontada pra finesine com offset)
extern fixed_t  finesine[5*FINEANGLES/4];
extern fixed_t* finecosine;
```

**Usar:**
```c
// Sem float, sem math.h, sem FPU
angle_t angle = playerAngle >> ANGLETOFINESHIFT;  // converte para índice de 13 bits
fixed_t sinVal = finesine[angle & FINEMASK];       // lookup direto
fixed_t cosVal = finecosine[angle & FINEMASK];     // lookup direto
```

**Lição:** Quando você não pode usar uma operação cara (float, trig), **pré-compute e indexe**.

---

## O Game Loop Real — 15 Linhas, Sem Gordura

```c
// d_main.c — o loop principal completo
void D_DoomLoop(void)
{
    if (demorecording)
        G_BeginRecording();

    I_InitGraphics();

    while (1)
    {
        // 1. I/O síncrono — sempre, sem skip
        I_StartFrame();

        // 2. Processar um ou mais ticks
        if (singletics)
        {
            I_StartTic();
            D_ProcessEvents();
            G_BuildTiccmd(&netcmds[consoleplayer][maketic%BACKUPTICS]);
            if (advancedemo)
                D_DoAdvanceDemo();
            M_Ticker();
            G_Ticker();
            gametic++;
            maketic++;
        }
        else
        {
            TryRunTics(); // roda pelo menos um tic
        }

        // 3. Som — sempre
        S_UpdateSounds(players[consoleplayer].mo);

        // 4. Display — sempre
        D_Display();

        // 5. Sound mixing — sempre
        I_UpdateSound();
        I_SubmitSound();
    }
}
```

**O que notar:**
- Sem `if (needsUpdate)` — executa sempre e inibe resultado se necessário
- `TryRunTics()` garante pelo menos um tick — nunca zera
- Som e display são chamados toda iteração, incondicionalmente
- O loop inteiro sem scroll é menor que qualquer função típica de "enterprise"

---

## Zone Memory Allocator — Controle Total

O DOOM não usa `malloc()` diretamente em nenhum lugar do engine. Tudo passa pelo Zone:

```c
// z_zone.h — a API completa do memory manager
void*   Z_Malloc(int size, int tag, void* user);
void    Z_Free(void* ptr);
void    Z_FreeTags(int lowtag, int hightag);
void    Z_ChangeTag(void* ptr, int tag);
int     Z_FreeMemory(void);

// Tags definem lifetime dos dados:
#define PU_STATIC       1   // nunca libera (engine data)
#define PU_SOUND        2   // até som ser parado
#define PU_MUSIC        3   // até música ser parada
#define PU_DAVE         4   // reservado (!)
#define PU_LEVEL        50  // libera quando o level termina  ← CHAVE
#define PU_LEVSPEC      51  // parte do level
#define PU_CACHE        100 // pode ser liberado quando precisar de memória

// Uso:
mobj_t* thing = Z_Malloc(sizeof(mobj_t), PU_LEVEL, NULL);
// Quando o level termina:
Z_FreeTags(PU_LEVEL, PU_LEVSPEC);  // libera TUDO do level de uma vez
```

**Por que isso é genius de portabilidade:**
- Em sistemas sem `malloc` (embedded), você só reimplementa `Z_Malloc`
- Memory leaks são impossíveis — `Z_FreeTags(PU_LEVEL, ...)` mata tudo
- Debug centralizado: um breakpoint em `Z_Malloc` captura toda alocação
- Comportamento idêntico em todas as plataformas

---

## Platform Abstraction — i_system.h

Toda interação com o OS passa por funções com prefixo `I_`. O engine nunca chama o OS diretamente:

```c
// i_system.h — o contrato com o mundo exterior
void    I_Init(void);
void    I_InitGraphics(void);
void    I_ShutdownGraphics(void);
void    I_StartFrame(void);         // início de cada frame
void    I_StartTic(void);           // processar eventos do OS
void    I_GetTime(void);            // relógio
void    I_Error(char* error, ...);  // error handling
void    I_Quit(void);               // shutdown limpo
void    I_UpdateNoBlit(void);       // atualizar sem blit
void    I_FinishUpdate(void);       // page flip / blit

// Para portar o DOOM para nova plataforma:
// Implemente essas ~10 funções. Pronto.
```

**Resultado:** o DOOM foi portado para:
- Linux, Windows, macOS
- Game Boy Advance (GBADoom)
- Nintendo DS, PSP, PS2
- ATMs, impressoras, osciloscópios
- Calculadoras Texas Instruments
- Câmeras digitais, babás eletrônicas
- Smart fridges, wristwatches

Todos implementaram as mesmas ~10 funções de `i_system.h`.

---

## Nomenclatura — Prefixos de Subsistema (C puro)

```c
// DOOM 1 — sem classes, prefixos indicam subsistema
// d_ = doom main / coordenação geral
D_DoomLoop()
D_DoomMain()
D_PostEvent()

// g_ = game logic
G_Ticker()
G_Responder()
G_BuildTiccmd()

// p_ = play (física, colisão, AI)
P_Ticker()
P_PlayerThink()
P_SpawnMobj()

// r_ = renderer
R_RenderPlayerView()
R_Init()
R_DrawViewBorder()

// i_ = interface com OS (platform layer)
I_GetTime()
I_FinishUpdate()
I_Error()

// z_ = zone memory
Z_Malloc()
Z_Free()
Z_FreeTags()

// w_ = WAD file system
W_CacheLumpName()
W_CheckNumForName()

// s_ = sound
S_StartSound()
S_UpdateSounds()

// m_ = menu / misc
M_Ticker()
M_Responder()
M_CheckParm()
```

Sistema de prefixos + C puro = namespace sem namespace. Legível, grep-ável, portável.

---

## O Event System — Queue Circular Simples

```c
// d_main.c — event handling sem biblioteca, sem framework
#define MAXEVENTS 64

event_t events[MAXEVENTS];  // buffer circular estático — sem malloc
int     eventhead;
int     eventtail;

// Produzir evento (chamado pelo i_system.c)
void D_PostEvent(event_t* ev)
{
    events[eventhead] = *ev;
    eventhead = (++eventhead) & (MAXEVENTS-1);  // wrap com bitmask
}

// Consumir eventos (chamado no game loop)
void D_ProcessEvents(void)
{
    event_t* ev;
    for (; eventtail != eventhead; eventtail = (++eventtail)&(MAXEVENTS-1))
    {
        ev = &events[eventtail];
        if (M_Responder(ev))
            continue;   // menu comeu o evento
        G_Responder(ev);
    }
}
```

**Lições:**
- Buffer estático — sem alocação dinâmica no hot path
- Wrap com bitmask (`& (MAXEVENTS-1)`) — mais rápido que módulo
- Produtor/consumidor sem mutex — game loop é single-threaded por design
- 64 eventos é suficiente para qualquer input humano por frame

---

## Diferenças Cruciais: DOOM 1 vs DOOM 3

```
DOOM 1 (1993)                      DOOM 3 (2004)
─────────────────────────────────  ─────────────────────────────────
C puro                             C++ com classes
Zero floats (fixed_t tudo)         Floats livres
Trig por lookup table              math.h + FPU
~30k linhas                        ~600k linhas
Platform layer = 10 funções        idLib completa
malloc wrappado em Z_Malloc        new/delete wrappado
Sem exceções (impossível em C)     Sem exceções (decisão)
Sem templates (impossível em C)    Templates mínimos
Roda em 4MB RAM                    Precisa de muito mais
Portável para qualquer coisa       Portável para desktops modernos
```

**Para a skill "roda em qualquer coisa":**
O DOOM 1 é a referência correta. O DOOM 3 é "bom código C++". São objetivos diferentes.

---

## O Princípio Mais Importante do DOOM 1

Carmack na nota de lançamento do código (1997):

> *"The code is quite portable, and it should be straightforward to bring it up on just about any platform."*

Isso não foi acidente. Foi resultado de:
1. **Zero dependências de OS no engine** — tudo via `i_system.h`
2. **Zero floats** — `fixed_t` roda em qualquer CPU
3. **Zero alocação dinâmica no engine** — Zone Allocator portável
4. **C puro** — qualquer compilador de qualquer plataforma
5. **Structs simples** — sem vtable, sem RTTI, sem surpresas ABI

**Quando alguém diz "código portável", deveria mirar aqui, não no DOOM 3.**
