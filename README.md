# 🔍 Reverse Engineering - Analyse de binaires ELF x86

Projet d'analyse et de rétro-ingénierie de binaires ELF 32-bit x86. Ce repository documente l'analyse complète de plusieurs exécutables, de la désassemblage à la reconstitution du code source C.

---

## 📋 Objectifs du projet

- Désassembler des exécutables compilés (ELF 32-bit)
- Identifier les structures du code assembleur (prologue, épilogue, calls, jumps)
- Comprendre le flux de contrôle des programmes
- Reconstituer le code source C original
- Extraire les informations sensibles (credentials hardcodés)

---

## 🛠️ Outils utilisés

### Identification du binaire
```bash
file ./app00
```
**Résultat :** Identifie le type de fichier (ELF 32-bit, architecture, linkage dynamique/statique, stripped ou non)

### Désassemblage
```bash
objdump -d -M intel ./app00
```
| Option | Description |
|--------|-------------|
| `-d` | Désassemble les sections exécutables |
| `-M intel` | Syntaxe Intel (plus lisible que AT&T) |

**Filtrer une fonction spécifique :**
```bash
objdump -d -M intel ./app00 | grep -A 100 "<main>:"
```

### Extraction des chaînes
```bash
strings ./app00
```
Extrait toutes les chaînes ASCII lisibles du binaire (passwords, messages, etc.)

### Extraction de sections spécifiques
```bash
objdump -s -j .rodata ./app00
```
Affiche le contenu hexadécimal de la section `.rodata` (read-only data : chaînes constantes)

---

## 📖 Méthodologie d'analyse

### Étape 1 : Identification
```bash
file ./binary
```
Détermine : architecture (32/64 bit), type (ELF/PE), linkage, si stripped

### Étape 2 : Extraction des chaînes
```bash
strings ./binary | head -30
```
Révèle souvent : passwords hardcodés, messages d'erreur, noms de fonctions

### Étape 3 : Désassemblage du main
```bash
objdump -d -M intel ./binary | grep -A 150 "<main>:"
```

### Étape 4 : Identification des structures

#### Prologue (début de fonction)
```asm
push   ebp           ; sauvegarde ancien base pointer
mov    ebp, esp      ; nouveau frame pointer  
sub    esp, 0xXX     ; réserve espace stack (variables locales)
```

#### Épilogue (fin de fonction)
```asm
mov    eax, 0x0      ; valeur de retour
leave                ; esp = ebp; pop ebp
ret                  ; retourne à l'appelant
```

#### Stack Canary (protection)
```asm
mov    eax, gs:0x14              ; charge le canary
mov    DWORD PTR [ebp-0x4], eax  ; sauvegarde sur stack
; ... code ...
mov    edx, DWORD PTR [ebp-0x4]  ; récupère canary
sub    edx, DWORD PTR gs:0x14    ; compare avec original
je     <ok>                       ; si égal → OK
call   __stack_chk_fail           ; sinon → abort
```

### Étape 5 : Analyse des calls
Identifier chaque `call` et ses arguments (pushés sur la stack en ordre inverse) :
```asm
push   0x804a008     ; arg1 : adresse string
call   printf        ; printf(arg1)
```

### Étape 6 : Analyse des jumps/conditions
```asm
test   eax, eax      ; teste si eax == 0
je     <addr>        ; jump if equal (ZF=1)
jne    <addr>        ; jump if not equal (ZF=0)
jl     <addr>        ; jump if less (signed)
jg     <addr>        ; jump if greater (signed)
```

### Étape 7 : Reconstitution du code C
Assembler toutes les informations pour reconstruire le code source.

---

## 🔐 Conventions d'appel x86 (cdecl)

### Passage d'arguments
Arguments pushés sur la stack de **droite à gauche** :
```c
strncmp(input, "password", 8);
```
```asm
push   0x8           ; arg3 : longueur
push   0x804a010     ; arg2 : "password"
push   eax           ; arg1 : input
call   strncmp
```

### Valeur de retour
Toujours dans le registre `EAX`

### Registres
| Registre | Usage |
|----------|-------|
| `EAX` | Valeur de retour, accumulateur |
| `EBX` | Base (callee-saved) |
| `ECX` | Compteur |
| `EDX` | Data |
| `ESP` | Stack pointer |
| `EBP` | Base pointer (frame) |
| `ESI/EDI` | Source/Destination index |

---

## 📁 Binaires analysés

| Fichier | Type | Description | Credentials |
|---------|------|-------------|-------------|
| [app00](./analyse_app00.md) | Password check | Vérification simple | `UtGmMu9O0dzse` |
| [app01](./analyse_app01.md) | Troll | Fake (affiche shrug) | Aucun |
| [app02](./analyse_app02.md) | Boucle | Affiche âges 0-66 | Aucun |
| [app03](./analyse_app03.md) | Password + canary | Comme app00 avec SSP | `fbx3key11dVw` |
| [app04](./analyse_app04.md) | Password obfusqué | Password sur stack | `gmk60B7pfoMJ` |
| [app05](./analyse_app05.md) | Password fragmenté | 3 getters + strncpy | `aEu38PrT5Zyo` |
| [app06](./analyse_app06.md) | MFA stdin | Username + Password + TOTP | `fdupont` / `irh0u803YdRB` / `817201` |
| [app07](./analyse_app07.md) | MFA argv | Comme app06 via CLI | `xdupont` / `rfG5HbNx2MJx` / `987109` |

---

## 📊 Évolution de la complexité

```
app00  →  app03  →  app04  →  app05  →  app06  →  app07
  │         │         │         │         │         │
  │         │         │         │         │         └─ Args CLI
  │         │         │         │         └─ Multi-facteurs (3 inputs)
  │         │         │         └─ Password fragmenté (3 getters)
  │         │         └─ Password construit sur stack
  │         └─ Stack canary protection
  └─ Password simple en .rodata
```

---

## 🧪 Exemple d'analyse rapide

```bash
# 1. Identifier le binaire
$ file app00
app00: ELF 32-bit LSB executable, Intel 80386, dynamically linked, not stripped

# 2. Chercher les strings intéressantes
$ strings app00 | grep -i pass
Password? 
UtGmMu9O0dzse

# 3. Désassembler main
$ objdump -d -M intel app00 | grep -A 50 "<main>:"

# 4. Identifier le strncmp et sa longueur
8049178:  6a 0d                 push   0xd          ; 13 caractères
804917a:  68 13 a0 04 08        push   0x804a013    ; "UtGmMu9O0dzse"
804917f:  ...
8049180:  e8 xx xx xx xx        call   strncmp

# 5. Tester
$ ./app00
Password? UtGmMu9O0dzse
Good!
```

---

## 📚 Ressources

- [x86 Assembly Guide](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html)
- [Intel x86 Instruction Set Reference](https://www.felixcloutier.com/x86/)
- [ELF Format Specification](https://refspecs.linuxfoundation.org/elf/elf.pdf)

---

