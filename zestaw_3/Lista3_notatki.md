# Lista 3 – DES: notatki i rozwiązania

Wszystkie bity trzymamy jako **stringi** złożone z `'0'` i `'1'`.

---

## Zadanie 1 – funkcje pomocnicze

### `bin2dec(bin_str)` – binarny string → int
Czytamy od lewej: wynik × 2 + bieżący bit. `'1001'` = 8 + 1 = 9.

```python
def bin2dec(bin_str):
    result = 0
    for b in bin_str:
        result = result * 2 + int(b)
    return result
```
(Krócej: `int(bin_str, 2)`.)

### `dec2bin(dec_str, pad)` – int → binarny string z zerami z przodu
Dzielimy przez 2 i zbieramy reszty, potem dopełniamy zerami do długości `pad`.
Uwaga: w zadaniu 5 wywołanie to `dec2bin(d, pad='8')`, więc `pad` może być stringiem – stąd `int(pad)`.

```python
def dec2bin(dec_str, pad):
    n = int(dec_str)
    if n == 0:
        bits = '0'
    else:
        bits = ''
        while n > 0:
            bits = str(n % 2) + bits
            n //= 2
    return bits.rjust(int(pad), '0')
```
(Krócej: `format(n, 'b').zfill(int(pad))`.)

### `shift_left(tab, n)` – przesunięcie cykliczne w lewo
Pierwsze `n` elementów idzie na koniec. Działa też na stringach (przyda się w zad. 2).

```python
def shift_left(tab, n):
    return tab[n:] + tab[:n]
```

### `permute(k, perm)` – permutacja według tablicy indeksów
`perm[i]` mówi, który znak z `k` trafia na pozycję `i`. Indeksy mogą się powtarzać
(permutacja rozszerzona, np. `E`: 32 → 48) albo być pominięte (np. `PC1`: 64 → 56).

```python
def permute(k, perm):
    return ''.join(k[i] for i in perm)
```

### `xor(bin_str1, bin_str2)` – XOR bit po bicie
W treści jest literówka: 1⊕0 = **1**. Jednakowe bity → `0`, różne → `1`.

```python
def xor(bin_str1, bin_str2):
    return ''.join('0' if a == b else '1' for a, b in zip(bin_str1, bin_str2))
```

---

## Zadanie 2 – generowanie podkluczy (key schedule)

1. `permute(key, PC1)`: 64 → 56 bitów (odrzucane są bity parzystości).
2. Podział na `left` i `right` po 28 bitów.
3. 16 razy: przesuń obie połówki o `shift_table[i]`, sklej, `permute(..., PC2)`: 56 → 48 bitów.

**Pułapka:** przesunięcia się **kumulują** – w każdej rundzie przesuwamy wynik poprzedniej
rundy (nadpisujemy `left`/`right`). Suma `shift_table` = 28, więc po 16 rundach połówki
wracają do stanu początkowego.

```python
def key_schedule(key):
    key56 = permute(key, PC1)              # 64 -> 56 bitów
    left, right = key56[:28], key56[28:]   # 2 x 28 bitów
    subkeys = []
    for s in shift_table:
        left = shift_left(left, s)         # przesunięcie kumuluje się
        right = shift_left(right, s)
        subkeys.append(permute(left + right, PC2))  # 56 -> 48 bitów
    return subkeys
```

---

## Zadanie 3 – sieć Feistela

W każdej z 16 rund:
```
L = L XOR F(R, podklucz_i)
zamiana L <-> R        (oprócz ostatniej rundy!)
```

**Dlaczego się odwraca, nawet gdy F nie jest odwracalna?** `R` przechodzi przez rundę
bez zmian, więc przy deszyfrowaniu można ponownie policzyć `F(R, k)` i zrobić XOR:
`(L ⊕ X) ⊕ X = L`. `F` nie trzeba odwracać, tylko przeliczyć (stąd test z `F2`,
która zawsze zwraca same jedynki).

```python
def Feistel(message, subkeys, F):
    left, right = message[:32], message[32:]
    for i, k in enumerate(subkeys):
        left = xor(left, F(right, k))
        if i < len(subkeys) - 1:          # w ostatniej rundzie bez zamiany
            left, right = right, left
    return left + right
```

Typowe błędy: zamiana także w 16. rundzie; podanie do `F` lewej połówki zamiast prawej.

---

## Zadanie 4 – funkcja F

```
32 bity --E--> 48 --XOR podklucz--> 48 --8 S-boxów--> 32 --P--> 32
```

1. Rozszerzenie `E`: 32 → 48 bitów.
2. XOR z podkluczem (jedyne miejsce, gdzie wchodzi klucz).
3. 8 grup po 6 bitów, grupa `i` → `SBox[i]`:
   - wiersz = bit pierwszy + ostatni (np. `'101011'` → `'11'` = 3),
   - kolumna = 4 środkowe bity (`'0101'` = 5),
   - wynik zapisujemy jako **4 bity**.
   S-boxy to jedyny nieliniowy element DES-a.
4. Permutacja `P`.

```python
def F(right, subkey):
    x = xor(permute(right, E), subkey)        # 1 + 2
    out = ''
    for i in range(8):                        # 3
        block = x[6*i:6*i+6]
        row = bin2dec(block[0] + block[5])
        col = bin2dec(block[1:5])
        out += dec2bin(SBox[i][row][col], 4)  # zawsze 4 bity!
    return permute(out, P)                    # 4
```

Uwaga: komórka z `E`, `SBox`, `P` jest pod definicją `F` – trzeba ją uruchomić przed testami.

Typowe błędy: brak dopełnienia do 4 bitów; pomylenie wiersza z kolumną; jeden S-box dla wszystkich grup.

---

## Zadanie 5 – cały DES i odszyfrowanie obrazka

```python
def DES(message, subkeys):
    m = permute(message, IP)          # 1. permutacja początkowa
    m = Feistel(m, subkeys, F)        # 2. 16 rund Feistela
    return permute(m, FP)             # 3. permutacja końcowa (odwrotność IP)
```

Deszyfrowanie: `DES(c, subkeys[::-1])`.

Uwagi do notebooka:
- `IP` i `FP` są zdefiniowane pod testami – uruchom tę komórkę wcześniej.
- W komórce wczytującej obrazek ostatnia linia `mg_enc` to literówka – zamień na `img_enc`.

Odszyfrowanie obrazka:

```python
subkeys = key_schedule(key)
rev = subkeys[::-1]
img_t = ''.join(DES(bits[i:i+64], rev) for i in range(0, len(bits), 64))

def split_img(s, n):
    return [s[i:i+n] for i in range(0, len(s), n)]
```

Potem gotowa komórka z `Image.fromarray(...)`. Obrazek: 400×540×3 = 648 000 bajtów,
dzieli się przez 8, więc bez paddingu. Odszyfrowanie trwa od kilkudziesięciu sekund do paru minut.

### Co to za obraz?
**Salvador Dalí, „Trwałość pamięci” („The Persistence of Memory”), 1931** – roztapiające się zegary.

Uwaga do sprawozdania: obrazek był szyfrowany w trybie **ECB** (każdy blok osobno, tym samym
kluczem). Te same bloki jawne dają te same szyfrogramy, więc w dużych jednolitych grafikach
wzory pozostają widoczne mimo szyfrowania.
