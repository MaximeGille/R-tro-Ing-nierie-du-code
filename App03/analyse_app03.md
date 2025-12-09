# Analyse du binaire `app03`

> **Type:** ELF 32-bit LSB executable, Intel 80386  
> **Linkage:** Dynamically linked  
> **Stripped:** No  
> **Protection:** Stack canary (SSP)

---

## 1. Prologue et Épilogue

### Prologue
```asm
push   ebp                    ; sauvegarde ancien base pointer
mov    ebp, esp               ; nouveau frame pointer
sub    esp, 0x28              ; réserve 40 bytes sur la stack
mov    eax, gs:0x14           ; charge le stack canary
mov    DWORD PTR [ebp-0x4], eax  ; sauvegarde le canary
xor    eax, eax               ; clear eax
```

### Épilogue
```asm
mov    eax, 0x0               ; return 0
mov    edx, DWORD PTR [ebp-0x4]  ; récupère le canary sauvegardé
sub    edx, DWORD PTR gs:0x14    ; compare avec l'original
je     8049237                   ; si égal → OK
call   __stack_chk_fail          ; sinon → stack smashing détecté
leave
ret
```

---

## 2. Calls et Arguments

| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x80491d2` | `printf` | `"Password? "` | Affiche le prompt |
| `0x80491e6` | `fgets` | `buffer`, `31`, `stdin` | Lit 31 chars max depuis stdin |
| `0x80491f9` | `strncmp` | `buffer`, `"fbx3key11dVw"`, `12` | Compare les 12 premiers chars |
| `0x804920a` | `puts` | `"Good!"` | Affiche si password correct |
| `0x8049219` | `puts` | `"Bad :("` | Affiche si password incorrect |

---

## 3. Jumps et Conditions

### Vérification du mot de passe
```asm
test   eax, eax      ; teste le retour de strncmp (0 = égal)
jne    8049214       ; si != 0 → saute vers "Bad :("
```

### Vérification du stack canary
```asm
sub    edx, DWORD PTR gs:0x14   ; compare canary actuel vs original
je     8049237                   ; si égal → sortie normale
call   __stack_chk_fail          ; sinon → abort (buffer overflow détecté)
```

### Flux de contrôle
```
strncmp() == 0 ?
       │
       ├── OUI → puts("Good!") → jmp vérif canary
       │
       └── NON → puts("Bad :(") → vérif canary
                                        │
                              canary intact ?
                                        │
                              ├── OUI → return 0
                              └── NON → __stack_chk_fail()
```

---

## 4. Fonctionnement général

Programme de **vérification de mot de passe** avec protection stack canary :

1. Initialise le stack canary (protection anti-overflow)
2. Affiche `"Password? "`
3. Lit l'entrée utilisateur (max 31 caractères)
4. Compare avec le mot de passe hardcodé
5. Affiche le résultat
6. Vérifie l'intégrité du stack canary avant de quitter

### Code C reconstitué

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv) {
    char buffer[32];
    printf("Password? ");
    fgets(buffer, 31, stdin);
    if (strncmp(buffer, "fbx3key11dVw", 12) == 0)
        puts("Good!");
    else
        puts("Bad :(");
    return 0;
}
```

---

## 5. Informations extraites

| Élément | Valeur |
|---------|--------|
| **Password** | `fbx3key11dVw` |
| **Taille buffer** | 32 bytes |
| **Taille comparaison** | 12 caractères |
| **Protection** | Stack canary (SSP) |

---

## 6. Différences avec app00

| Aspect | app00 | app03 |
|--------|-------|-------|
| Stack canary | Non | Oui (`gs:0x14`) |
| Password | `UtGmMu9O0dzse` | `fbx3key11dVw` |
| Longueur comparée | 13 | 12 |
| Taille stack | 0x20 (32) | 0x28 (40) |
