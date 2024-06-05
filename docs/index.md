# Introduzione 
La programmazione low-level può spaventare data la sua difficoltà. 
Con questa guida, voglio togliere tutti i dubbi a riguardo. Partirò dalle basi, non lasciando nulla per scontato, e con molti approfondimenti low-level. L'obiettivo di questa documentazione è quello di scrivere un emulatore del chip-8 con C++.
## Requisiti 
È consigliato avere delle conoscenze dei componenti di un PC ed è necessario conoscere le basi di C++.
## Cos'è il Chip-8
Il chip-8 in realtà non è una macchina fisica, ma bensì una macchina virtuale. Questo rende la scrittura dell'emulatore più semplice rispetto, per esempio, a quella di un Game Boy, poiché non dovremo emulare hardware fisico. Di conseguenza, questo emulatore può essere considerato come un interprete, il quale avrà il ruolo di interpretare i programmi (ROM) scritti per l'architettura del chip-8.
## Come funziona la CPU
È fondamentale comprendere come funziona la CPU per poter capire appieno l'emulatore. La CPU legge istruzioni dalla memoria (RAM e ROM). Prima, le istruzioni del gioco contenute nella memoria ROM vengono passate alla memoria RAM. Infine, la CPU prende le informazioni dalla RAM e le esegue. La CPU ha bisogno di istruzioni ben precise, quindi è cruciale scrivere correttamente e in modo accurato l'emulatore.
## Cosa fa un emulatore
Un emulatore legge le istruzioni in codice macchina che sono state scritte per la specifica console. Dopo le interpreta e poi replica le funzionalità della macchina che si sta emulando sulla nostra. Le ROM contengono le istruzioni, l'emulatore le legge e successivamente emula la macchina originale.
## Componenti del Chip-8
### 16 registri 8-bit 
I registri sono una parte dedicata allo storage, ma solo per operazioni temporanee (pochi KB di spazio), mentre i dati a lungo termine sono contenuti in una memoria di massa come SSD o HDD. In questo caso, le operazioni sono dei caricamenti di dati dalla memoria nei registri, operare nei registri e successivamente immagazzinare il risultato nella memoria. I registri del Chip-8 vanno dal V0 al VF. Ogni registro può avere un valore da 0x00 a 0xFF.
### 4K Bytes di memoria RAM
I registri hanno troppo poco spazio per mantenere dati a lungo termine. Qui entra in gioco la RAM, la quale nel caso del chip-8 funge sia da magazzino per le istruzioni del programma in esecuzione, sia per mantenere dati a lungo termine. Per accedere alle locazioni specifiche della memoria RAM, la CPU usa gli indirizzi. Infatti, ogni locazione di memoria ha un indirizzo specifico per poter accedervi. Avendo 4K byte di memoria, lo spazio degli indirizzi di memoria va da 0x000 a 0xFFF.
### Registro index 16-bit
Viene usato per immagazzinare degli indirizzi di memoria da usare nelle operazioni. Il massimo valore degli indirizzi di memoria è 0xFFF.
### 16-bit Program Counter
È un registro che tiene gli indirizzi di memoria della successiva istruzione da eseguire.
### 16-level Stack
Uno stack viene usato dalla CPU per tenere traccia dell'ordine di esecuzione quando chiama delle funzioni. Il chip-8 ha 16 livelli di stack, questo implica che possono essere memorizzati 16 valori del Program Counter (PC).
### 8-bit Timer
Il Chip-8 ha un timer integrato. Se gli viene caricato un valore, si decrementerà ad una frequenza di 60Hz. 
### 8-bit Timer del suono
È un altro timer utilizzato per i suoni. Funziona come l'altro timer, ovvero si decrementa a 60Hz se non è zero.
### 16 Input Keys
Il Chip-8 ha 16 tasti per l'input, i quali corrispondono ai primi 16 valori esadecimali.
### Memoria Display Monocromatico 64x32 Pixel
Il chip-8 ha un'ulteriore memoria dedicata al memorizzare i grafici da visualizzare. Rappresenta uno schermo di 64x32 pixel. Possono essere visualizzati solo due colori, ovvero il bianco e il nero. Per disegnare a schermo, il Chip-8 usa l'operazione XOR.
## Iniziamo a scrivere il codice 
### Definiamo i membri della classe
Qui abbiamo tutti i componenti che ho spiegato in precedenza.
```
`
``#include <cstdint>`

`class Chip8 {`
`public:`
    `uint8_t registers[16]{};       // 16 registri V0-VF`
    `uint8_t memory[4096]{};        // Memoria di 4096 byte`
    `uint16_t index{};              // Puntatore di indice`
    `uint16_t pc{};                 // Contatore di programma`
    `uint16_t stack[16]{};          // Stack`
    `uint8_t sp{};                  // Puntatore dello stack`
    `uint8_t delayTimer{};          // Timer di ritardo`
    `uint8_t soundTimer{};          // Timer del suono`
    `uint8_t keypad[16]{};          // Tastiera`
    `uint32_t video[64 * 32]{};     // Buffer video`
    `uint16_t opcode;               // Opcode corrente`
`};`
```
## Caricare i fonts

I caratteri sono rappresentati come sprite e ogni sprite è costituito da 5 byte. 

```cpp
const unsigned int FONTSET_SIZE = 80;

uint8_t fontset[FONTSET_SIZE] =
{
	0xF0, 0x90, 0x90, 0x90, 0xF0, // 0
	0x20, 0x60, 0x20, 0x20, 0x70, // 1
	0xF0, 0x10, 0xF0, 0x80, 0xF0, // 2
	0xF0, 0x10, 0xF0, 0x10, 0xF0, // 3
	0x90, 0x90, 0xF0, 0x10, 0x10, // 4
	0xF0, 0x80, 0xF0, 0x10, 0xF0, // 5
	0xF0, 0x80, 0xF0, 0x90, 0xF0, // 6
	0xF0, 0x10, 0x20, 0x40, 0x40, // 7
	0xF0, 0x90, 0xF0, 0x90, 0xF0, // 8
	0xF0, 0x90, 0xF0, 0x10, 0xF0, // 9
	0xF0, 0x90, 0xF0, 0x90, 0x90, // A
	0xE0, 0x90, 0xE0, 0x90, 0xE0, // B
	0xF0, 0x80, 0x80, 0x80, 0xF0, // C
	0xE0, 0x90, 0x90, 0x90, 0xE0, // D
	0xF0, 0x80, 0xF0, 0x80, 0xF0, // E
	0xF0, 0x80, 0xF0, 0x80, 0x80  // F
};
```
### Per caricare la memoria
```
const unsigned int FONTSET_START_ADDRESS = 0x50;

Chip8::Chip8()
{
	[...]

	// Load fonts into memory
	for (unsigned int i = 0; i < FONTSET_SIZE; ++i)
	{
		memory[FONTSET_START_ADDRESS + i] = fontset[i];
	}
}
```
## Caricare una ROM
Questa parte di codice serve a caricare una ROM da far eseguire al Chip-8
```
`#include <fstream>`

`const unsigned int START_ADDRESS = 0x200;`

`void Chip8::LoadROM(char const* filename) {`

    `std::ifstream file(filename, std::ios::binary | std::ios::ate);`
    
    `if (file.is_open()) {`
        `std::streampos size = file.tellg();`
        `char* buffer = new char[size];`
        `file.seekg(0, std::ios::beg);`
        `file.read(buffer, size);`
        `file.close();`
        `for (long i = 0; i < size; ++i) {`
            `memory[START_ADDRESS + i] = buffer[i];`
        `}`
        
        `delete[] buffer;`
    `}`
`}`
```
## Le istruzioni
Il chip-8 ha 34 istruzioni che dobbiamo emulare.
- **0nnn - SYS addr**
- **00E0 - CLS**
- **00EE - RET**
- **1nnn - JP addr**
- **2nnn - CALL addr**
- **3xkk - SE Vx, byte**
- **4xkk - SNE Vx, byte**
- **5xy0 - SE Vx, Vy**
- **6xkk - LD Vx, byte**
- **7xkk - ADD Vx, byte**
- **8xy0 - LD Vx, Vy**
- **8xy1 - OR Vx, Vy**
- **8xy2 - AND Vx, Vy**
- **8xy3 - XOR Vx, Vy**
- **8xy4 - ADD Vx, Vy**
- **8xy5 - SUB Vx, Vy**
- **8xy6 - SHR Vx {, Vy}**
- **8xy7 - SUBN Vx, Vy**
- **8xyE - SHL Vx {, Vy}**
- **9xy0 - SNE Vx, Vy**
- **Annn - LD I, addr**
- **Bnnn - JP V0, addr**
- **Cxkk - RND Vx, byte**
- **Dxyn - DRW Vx, Vy, nibble**
- **Ex9E - SKP Vx**
- **ExA1 - SKNP Vx**
- **Fx07 - LD Vx, DT**
- **Fx0A - LD Vx, K**
- **Fx15 - LD DT, Vx**
- **Fx18 - LD ST, Vx**
- **Fx1E - ADD I, Vx**
- **Fx29 - LD F, Vx**
- **Fx33 - LD B, Vx**
- **Fx55 - LD [I], Vx**
- **Fx65 - LD Vx, [I]

## Fetch, Decode, Execute
Il ciclo che effettua una CPU nel contesto di CHIP-8 è divisa in tre fasi principali: **fetch**, **decode** ed **execute**.  Vediamole:
### Fetch
Durante la fase di fetch, l'emulatore legge la prossima istruzione dalla memoria.
La seguente riga di codice realizza questa operazione:
`opcode = (memory[pc] << 8u) | memory[pc + 1];
### Decode
Nella fase di decodifica, l'istruzione prelevata viene interpretata per determinare quale operazione eseguire. La seguente riga di codice realizza questa operazione:
`((*this).*(table[(opcode & 0xF000u) >> 12u]))();`
### Execute
Durante la fase di esecuzione, la funzione individuata nella fase di decodifica viene chiamata per eseguire l'istruzione.
## SDL
SDL è una libreria che viene usata per renderizzare e ottenere gli input.
class Platform
```
`{`
`public:`
    `Platform(char const* title, int windowWidth, int windowHeight, int textureWidth, int textureHeight)`
    `{`
        `SDL_Init(SDL_INIT_VIDEO);`

        `window = SDL_CreateWindow(title, 0, 0, windowWidth, windowHeight, SDL_WINDOW_SHOWN);`

        `renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED);`

        `texture = SDL_CreateTexture(`
            `renderer, SDL_PIXELFORMAT_RGBA8888, SDL_TEXTUREACCESS_STREAMING, textureWidth, textureHeight);`
    `}`

    `~Platform()`
    `{`
        `SDL_DestroyTexture(texture);`
        `SDL_DestroyRenderer(renderer);`
        `SDL_DestroyWindow(window);`
        `SDL_Quit();`
    `}`

    `void Update(void const* buffer, int pitch)`
    `{`
        `SDL_UpdateTexture(texture, nullptr, buffer, pitch);`
        `SDL_RenderClear(renderer);`
        `SDL_RenderCopy(renderer, texture, nullptr, nullptr);`
        `SDL_RenderPresent(renderer);`
    `}`

    `bool ProcessInput(uint8_t* keys)`
    `{`
        `bool quit = false;`

        `SDL_Event event;`

        `while (SDL_PollEvent(&event))`
        `{`
            `switch (event.type)`
            `{`
                `case SDL_QUIT:`
                `{`
                    `quit = true;`
                `} break;`

                `case SDL_KEYDOWN:`
                `{`
                    `switch (event.key.keysym.sym)`
                    `{`
                        `case SDLK_ESCAPE:`
                        `{`
                            `quit = true;`
                        `} break;`

                        `case SDLK_x:`
                        `{`
                            `keys[0] = 1;`
                        `} break;`
                        `// Aggiungi altre mappature dei tasti qui`
                    `}`
                `} break;`

                `case SDL_KEYUP:`
                `{`
                    `switch (event.key.keysym.sym)`
                    `{`
                        `case SDLK_x:`
                        `{`
                            `keys[0] = 0;`
                        `} break;`
                        `// Aggiungi altre mappature dei tasti qui`
                    `}`
                `} break;`
            `}`
        `}`

        `return quit;`
    `}`

`private:`
    `SDL_Window* window{};`
    `SDL_Renderer* renderer{};`
    `SDL_Texture* texture{};`
`};`
```
## Loop Principale
Il cuore dell'emulatore. Questo ciclo continua a eseguire le istruzioni fino a quando l'utente decide di uscire. 

```cpp

#include <iostream>
#include <GL/glut.h>
#include "chip8.h"

chip8 myChip8;

int main(int argc, char **argv) 
{
    setupGraphics();
    setupInput();

    myChip8.initialize();
    myChip8.loadGame("pong");

    for(;;)
    {
        myChip8.emulateCycle();

        if(myChip8.drawFlag)
            drawGraphics();

        myChip8.setKeys();    
    }

    return 0;
}

```
## Fonti
https://tobiasvl.github.io/blog/write-a-chip-8-emulator/

https://austinmorlan.com/posts/chip8_emulator/#function-pointer-table

https://multigesture.net/articles/how-to-write-an-emulator-chip-8-interpreter/
