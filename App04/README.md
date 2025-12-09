# Analyse du binaire `app04`

> **Type:** ELF 32-bit LSB executable, Intel 80386  
> **Linkage:** Dynamically linked  
> **Stripped:** No  
> **Protection:** Stack canary (SSP)

---

## 1. Prologue et Épilogue

### main() - Prologue
```asm
push   ebp
mov    ebp, esp
sub    esp, 0x28              ; réserve 40 bytes
mov    eax, gs:0x14           ; charge stack canary
mov    DWORD PTR [ebp-0x4], eax
xor    eax, eax
```

### main() - Épilogue
```asm
mov    eax, 0x0
mov    edx, DWORD PTR [ebp-0x4]
sub    edx, DWORD PTR gs:0x14  ; vérifie canary
je     8049294
call   __stack_chk_fail
leave
ret
```

### check_password() - Prologue
```asm
push   ebp
mov    ebp, esp
sub    esp, 0x18              ; réserve 24 bytes
mov    eax, gs:0x14           ; charge stack canary
mov    DWORD PTR [ebp-0x4], eax
xor    eax, eax
```

### check_password() - Épilogue
```asm
mov    edx, DWORD PTR [ebp-0x4]
sub    edx, DWORD PTR gs:0x14
je     8049218
call   __stack_chk_fail
leave
ret
```

---

## 2. Calls et Arguments

### main()
| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x8049236` | `printf` | `"Password? "` | Affiche le prompt |
| `0x804924a` | `fgets` | `buffer`, `31`, `stdin` | Lit 31 chars max |
| `0x8049256` | `check_password` | `buffer` | Vérifie le mot de passe |
| `0x8049267` | `puts` | `"Good!"` | Si password correct |
| `0x8049276` | `puts` | `"Bad :("` | Si password incorrect |

### check_password()
| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x80491ef` | `strncmp` | `input`, `password_local`, `12` | Compare 12 chars |

---

## 3. Jumps et Conditions

### main()
```asm
test   eax, eax      ; teste retour de check_password
je     8049271       ; si == 0 → "Bad :("
                     ; sinon → "Good!"
```

### check_password()
```asm
test   eax, eax      ; teste retour de strncmp
jne    8049202       ; si != 0 → return 0 (échec)
mov    eax, 0x1      ; sinon → return 1 (succès)
```

### Flux de contrôle
```
main()
   │
   ▼
printf("Password? ")
   │
   ▼
fgets(buffer)
   │
   ▼
check_password(buffer)
   │
   ├── return 1 → puts("Good!")
   │
   └── return 0 → puts("Bad :(")


check_password(input)
   │
   ▼
Construit password sur la stack
   │
   ▼
strncmp(input, password, 12)
   │
   ├── == 0 → return 1
   │
   └── != 0 → return 0
```

---

## 4. Construction du mot de passe

Le mot de passe est construit **sur la stack** (pas en .rodata) :

```asm
mov    DWORD PTR [ebp-0x11], 0x366b6d67  ; "gmk6" (little endian)
mov    DWORD PTR [ebp-0xd],  0x70374230  ; "0B7p" (little endian)
mov    DWORD PTR [ebp-0x9],  0x4a4d6f66  ; "foMJ" (little endian)
mov    BYTE PTR [ebp-0x5],   0x0         ; null terminator
```

| Offset | Hex | ASCII (little endian) |
|--------|-----|----------------------|
| `[ebp-0x11]` | `0x366b6d67` | `gmk6` |
| `[ebp-0xd]` | `0x70374230` | `0B7p` |
| `[ebp-0x9]` | `0x4a4d6f66` | `foMJ` |
| `[ebp-0x5]` | `0x00` | `\0` |

**Password reconstitué:** `gmk60B7pfoMJ`

---

## 5. Fonctionnement général

Programme de **vérification de mot de passe** avec :
- Fonction séparée `check_password()`
- Password construit dynamiquement sur la stack (obfuscation légère)
- Double protection stack canary (main + check_password)

### Code C reconstitué

```c
#include <stdio.h>
#include <string.h>

int check_password(char *input) {
    char password[13] = "gmk60B7pfoMJ";
    if (strncmp(input, password, 12) == 0)
        return 1;
    return 0;
}

int main(int argc, char **argv) {
    char buffer[32];
    printf("Password? ");
    fgets(buffer, 31, stdin);
    if (check_password(buffer))
        puts("Good!");
    else
        puts("Bad :(");
    return 0;
}
```

---

## 6. Informations extraites

| Élément | Valeur |
|---------|--------|
| **Password** | `gmk60B7pfoMJ` |
| **Taille comparaison** | 12 caractères |
| **Fonctions** | `main`, `check_password` |
| **Protection** | Stack canary (×2) |
| **Obfuscation** | Password construit sur stack |

---

## 7. Évolution par rapport aux apps précédentes

| Aspect | app00/03 | app04 |
|--------|----------|-------|
| Password stockage | .rodata (visible avec `strings`) | Stack (construit dynamiquement) |
| Architecture | Tout dans main | Fonction `check_password` séparée |
| Logique retour | strncmp == 0 → Good | check_password retourne 1 → Good |
