# Analyse du binaire `app05`

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
sub    esp, 0x28
mov    eax, gs:0x14
mov    DWORD PTR [ebp-0x4], eax
xor    eax, eax
```

### main() - Épilogue
```asm
mov    eax, 0x0
mov    edx, DWORD PTR [ebp-0x4]
sub    edx, DWORD PTR gs:0x14
je     80492fa
call   __stack_chk_fail
leave
ret
```

### get_p1/get_p2/get_p3 - Prologue/Épilogue minimal
```asm
push   ebp
mov    ebp, esp
mov    eax, <adresse_string>   ; retourne pointeur vers string
pop    ebp
ret
```

---

## 2. Calls et Arguments

### main()
| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x804929c` | `printf` | `"Password? "` | Affiche le prompt |
| `0x80492b0` | `fgets` | `buffer`, `31`, `stdin` | Lit 31 chars max |
| `0x80492bc` | `check_password` | `buffer` | Vérifie le mot de passe |
| `0x80492cd` | `puts` | `"Good!"` | Si password correct |
| `0x80492dc` | `puts` | `"Bad :("` | Si password incorrect |

### check_password()
| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x80491fb` | `get_p1` | - | Retourne `"aEu3"` |
| `0x8049203` | `get_p2` | - | Retourne `"8PrT"` |
| `0x804920b` | `get_p3` | - | Retourne `"5Zyo"` |
| `0x804921c` | `strncpy` | `dest`, `p1`, `4` | Copie partie 1 |
| `0x8049230` | `strncpy` | `dest+4`, `p2`, `4` | Copie partie 2 |
| `0x8049244` | `strncpy` | `dest+8`, `p3`, `4` | Copie partie 3 |
| `0x8049255` | `strncmp` | `input`, `password`, `12` | Compare 12 chars |

### get_p1(), get_p2(), get_p3()
| Fonction | Retourne | Adresse | Valeur |
|----------|----------|---------|--------|
| `get_p1` | `0x804a008` | .rodata | `"aEu3"` |
| `get_p2` | `0x804a00d` | .rodata | `"8PrT"` |
| `get_p3` | `0x804a012` | .rodata | `"5Zyo"` |

---

## 3. Jumps et Conditions

### check_password()
```asm
test   eax, eax      ; teste retour de strncmp
jne    8049268       ; si != 0 → return 0
mov    eax, 0x1      ; sinon → return 1
```

### main()
```asm
test   eax, eax      ; teste retour de check_password
je     80492d7       ; si == 0 → "Bad :("
                     ; sinon → "Good!"
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
   │    ┌─────────────────────────────────┐
   │    │  p1 = get_p1() → "aEu3"        │
   │    │  p2 = get_p2() → "8PrT"        │
   │    │  p3 = get_p3() → "5Zyo"        │
   │    │                                 │
   │    │  strncpy(pass, p1, 4)          │
   │    │  strncpy(pass+4, p2, 4)        │
   │    │  strncpy(pass+8, p3, 4)        │
   │    │                                 │
   │    │  pass = "aEu38PrT5Zyo"         │
   │    │                                 │
   │    │  strncmp(input, pass, 12)      │
   │    └─────────────────────────────────┘
   │
   ├── return 1 → puts("Good!")
   │
   └── return 0 → puts("Bad :(")
```

---

## 4. Construction du mot de passe

Le password est **fragmenté** en 3 parties stockées séparément dans `.rodata` :

| Partie | Fonction | Adresse | Valeur |
|--------|----------|---------|--------|
| 1 | `get_p1()` | `0x804a008` | `aEu3` |
| 2 | `get_p2()` | `0x804a00d` | `8PrT` |
| 3 | `get_p3()` | `0x804a012` | `5Zyo` |

**Assemblage dans check_password():**
```
[ebp-0x10]      [ebp-0xc]       [ebp-0x8]
    │               │               │
    ▼               ▼               ▼
┌───────┬───────┬───────┬───┐
│ aEu3  │ 8PrT  │ 5Zyo  │\0 │
└───────┴───────┴───────┴───┘
   4B      4B      4B     1B
```

**Password reconstitué:** `aEu38PrT5Zyo`

---

## 5. Fonctionnement général

Programme de **vérification de mot de passe** avec :
- Password fragmenté en 3 parties (obfuscation)
- 3 fonctions getter pour récupérer chaque fragment
- Assemblage dynamique via `strncpy`

### Code C reconstitué

```c
#include <stdio.h>
#include <string.h>

char* get_p1() { return "aEu3"; }
char* get_p2() { return "8PrT"; }
char* get_p3() { return "5Zyo"; }

int check_password(char *input) {
    char password[13];
    char *p1 = get_p1();
    char *p2 = get_p2();
    char *p3 = get_p3();
    
    strncpy(password, p1, 4);
    strncpy(password + 4, p2, 4);
    strncpy(password + 8, p3, 4);
    password[12] = '\0';
    
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
| **Password** | `aEu38PrT5Zyo` |
| **Taille comparaison** | 12 caractères |
| **Fonctions** | `main`, `check_password`, `get_p1`, `get_p2`, `get_p3` |
| **Protection** | Stack canary |
| **Obfuscation** | Password fragmenté en 3 parties |

---

## 7. Évolution par rapport aux apps précédentes

| Aspect | app04 | app05 |
|--------|-------|-------|
| Stockage password | Stack (mov directs) | .rodata (fragmenté) |
| Fonctions | 2 (`main`, `check_password`) | 5 (+`get_p1/p2/p3`) |
| Assemblage | Instructions `mov` | Appels `strncpy` |
| Visibilité strings | Partiellement masqué | Visible mais fragmenté |
