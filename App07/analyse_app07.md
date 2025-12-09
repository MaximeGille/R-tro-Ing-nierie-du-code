# Analyse du binaire `app07`

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
sub    esp, 0x50              ; réserve 80 bytes
mov    eax, DWORD PTR [ebp+0xc]  ; argv
mov    DWORD PTR [ebp-0x50], eax
mov    eax, gs:0x14
mov    DWORD PTR [ebp-0x4], eax
xor    eax, eax
```

### main() - Épilogue
```asm
mov    eax, 0x0
mov    edx, DWORD PTR [ebp-0x4]
sub    edx, DWORD PTR gs:0x14
je     804932a
call   __stack_chk_fail
leave
ret
```

### check_username/check_password/check_totp - Prologue/Épilogue minimal
```asm
push   ebp
mov    ebp, esp
; ... comparaison ...
leave
ret
```

---

## 2. Calls et Arguments

### main()
| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x8049253` | `printf` | `"Usage: %s USERNAME PASSWORD TOTP\n"`, `argv[0]` | Message d'erreur si argc != 4 |
| `0x8049274` | `strncpy` | `username_buf`, `argv[1]`, `32` | Copie username |
| `0x804928b` | `strncpy` | `password_buf`, `argv[2]`, `32` | Copie password |
| `0x80492a2` | `strncpy` | `totp_buf`, `argv[3]`, `8` | Copie TOTP |
| `0x80492ae` | `check_username` | `username_buf` | Vérifie username |
| `0x80492be` | `check_password` | `password_buf` | Vérifie password |
| `0x80492ce` | `check_totp` | `totp_buf` | Vérifie TOTP |
| `0x80492df` | `puts` | `"Good!"` | Succès total |
| `0x80492ee` | `puts` | `"Bad TOTP :("` | Échec TOTP |
| `0x80492fd` | `puts` | `"Bad password :("` | Échec password |
| `0x804930c` | `puts` | `"Bad username :("` | Échec username |

### Fonctions de vérification
| Fonction | Comparaison | Longueur | Valeur attendue |
|----------|-------------|----------|-----------------|
| `check_username` | `strncmp("xdupont", input, 7)` | 7 | `xdupont` |
| `check_password` | `strncmp("rfG5HbNx2MJx", input, 12)` | 12 | `rfG5HbNx2MJx` |
| `check_totp` | `strncmp("987109", input, 6)` | 6 | `987109` |

---

## 3. Jumps et Conditions

### Vérification argc
```asm
cmp    DWORD PTR [ebp+0x8], 0x4   ; argc == 4 ?
je     8049265                     ; si oui → continue
; sinon → affiche usage et return -1
printf("Usage: %s USERNAME PASSWORD TOTP\n", argv[0])
mov    eax, 0xffffffff             ; return -1
jmp    8049319
```

### Structure imbriquée (3 niveaux)
```asm
; Vérification username
call   check_username
test   eax, eax
je     8049307            ; → "Bad username :("

; Vérification password
call   check_password
test   eax, eax
je     80492f8            ; → "Bad password :("

; Vérification TOTP
call   check_totp
test   eax, eax
je     80492e9            ; → "Bad TOTP :("

; Succès
puts("Good!")
```

### Flux de contrôle
```
                  argc == 4 ?
                      │
            ┌────NO───┴───YES────┐
            │                    │
            ▼                    ▼
     printf(Usage)        strncpy(argv[1] → username)
     return -1            strncpy(argv[2] → password)
                          strncpy(argv[3] → totp)
                                 │
                                 ▼
                     check_username() == 1 ?
                                 │
                       ┌────NO───┴───YES────┐
                       │                    │
                       ▼                    ▼
               "Bad username :("    check_password() == 1 ?
                                            │
                                  ┌────NO───┴───YES────┐
                                  │                    │
                                  ▼                    ▼
                          "Bad password :("    check_totp() == 1 ?
                                                       │
                                             ┌────NO───┴───YES────┐
                                             │                    │
                                             ▼                    ▼
                                      "Bad TOTP :("           "Good!"
```

---

## 4. Variables locales (main)

| Offset | Taille | Usage |
|--------|--------|-------|
| `[ebp+0x8]` | 4 bytes | `argc` |
| `[ebp+0xc]` | 4 bytes | `argv` |
| `[ebp-0x44]` | 32 bytes | Buffer username |
| `[ebp-0x24]` | 32 bytes | Buffer password |
| `[ebp-0x4c]` | 8 bytes | Buffer TOTP |
| `[ebp-0x4]` | 4 bytes | Stack canary |

---

## 5. Fonctionnement général

Programme d'**authentification MFA via arguments CLI** :
- Vérifie que 3 arguments sont passés
- Copie les arguments dans des buffers locaux
- Vérifie username, password, TOTP

### Usage
```bash
./app07 USERNAME PASSWORD TOTP
./app07 xdupont rfG5HbNx2MJx 987109
```

### Code C reconstitué

```c
#include <stdio.h>
#include <string.h>

int check_username(char *input) {
    if (strncmp("xdupont", input, 7) == 0)
        return 1;
    return 0;
}

int check_password(char *input) {
    if (strncmp("rfG5HbNx2MJx", input, 12) == 0)
        return 1;
    return 0;
}

int check_totp(char *input) {
    if (strncmp("987109", input, 6) == 0)
        return 1;
    return 0;
}

int main(int argc, char **argv) {
    char username[32];
    char password[32];
    char totp[8];
    
    if (argc != 4) {
        printf("Usage: %s USERNAME PASSWORD TOTP\n", argv[0]);
        return -1;
    }
    
    strncpy(username, argv[1], 32);
    strncpy(password, argv[2], 32);
    strncpy(totp, argv[3], 8);
    
    if (check_username(username)) {
        if (check_password(password)) {
            if (check_totp(totp)) {
                puts("Good!");
            } else {
                puts("Bad TOTP :(");
            }
        } else {
            puts("Bad password :(");
        }
    } else {
        puts("Bad username :(");
    }
    return 0;
}
```

---

## 6. Informations extraites

| Élément | Valeur |
|---------|--------|
| **Username** | `xdupont` |
| **Password** | `rfG5HbNx2MJx` |
| **TOTP** | `987109` |
| **Fonctions** | `main`, `check_username`, `check_password`, `check_totp` |
| **Protection** | Stack canary |
| **Input** | Arguments CLI (argv) |

---

## 7. Évolution par rapport à app06

| Aspect | app06 | app07 |
|--------|-------|-------|
| Input | `stdin` (fgets) | CLI args (argv) |
| Prompts | `"Username?"`, `"Password?"`, `"TOTP?"` | Aucun |
| Validation argc | Non | Oui (argc == 4) |
| Message erreur | Non | `"Usage: %s USERNAME PASSWORD TOTP"` |
| Copie données | fgets direct | strncpy depuis argv |
| Credentials | `fdupont` / `irh0u803YdRB` / `817201` | `xdupont` / `rfG5HbNx2MJx` / `987109` |
