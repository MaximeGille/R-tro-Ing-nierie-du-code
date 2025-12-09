# Analyse du binaire `app00`

> **Type:** ELF 32-bit LSB executable, Intel 80386  
> **Linkage:** Dynamically linked  
> **Stripped:** No

---

## 1. Prologue et Épilogue

### Prologue
```asm
push   ebp           ; sauvegarde ancien base pointer
mov    ebp, esp      ; nouveau frame pointer
sub    esp, 0x20     ; réserve 32 bytes sur la stack (buffer local)
```

### Épilogue
```asm
mov    eax, 0x0      ; return 0
leave                ; esp = ebp; pop ebp
ret
```

---

## 2. Calls et Arguments

| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x8049191` | `printf` | `"Password? "` | Affiche le prompt |
| `0x80491a5` | `fgets` | `buffer`, `31`, `stdin` | Lit 31 chars max depuis stdin |
| `0x80491b8` | `strncmp` | `buffer`, `"UtGmMu9O0dzse"`, `13` | Compare les 13 premiers chars |
| `0x80491c9` | `puts` | `"Good!"` | Affiche si password correct |
| `0x80491d8` | `puts` | `"Bad :("` | Affiche si password incorrect |

---

## 3. Jumps et Conditions

```asm
test   eax, eax      ; teste le retour de strncmp (0 = égal)
jne    80491d3       ; si != 0 → saute vers "Bad :("
```

### Flux de contrôle

```
strncmp() == 0 ?
       │
       ├── OUI → puts("Good!") → jmp fin
       │
       └── NON → puts("Bad :(") → fin
```

---

## 4. Fonctionnement général

Programme de **vérification de mot de passe** :

1. Affiche `"Password? "`
2. Lit l'entrée utilisateur (max 31 caractères)
3. Compare avec le mot de passe hardcodé
4. Affiche le résultat

### Code C reconstitué

```c
#include <stdio.h>
#include <string.h>

int main() {
    char buffer[32];
    printf("Password? ");
    fgets(buffer, 31, stdin);
    if (strncmp(buffer, "UtGmMu9O0dzse", 13) == 0)
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
| **Password** | `UtGmMu9O0dzse` |
| **Taille buffer** | 32 bytes |
| **Taille comparaison** | 13 caractères |
