# DCL-pro-F1592

# Komplexní shrnutí řešení autorizace KFZKZ pro Fiori Asset Worklist

Tento dokument poskytuje **detailní popis** celého řešení autorizace podle hodnoty **KFZKZ** (v systému reprezentované jako *VehicleLicensePlateNumber*), které slouží jako identifikátor **budovy/objektu** vycházejícího z číselníku `ZAM_OBJEKT`.

Je určen:
- **vývojářům** – aby přesně věděli, jak řešení funguje a jak jej reprodukovat,
- **business** – aby byla srozumitelně popsána logika, účel a dopady.

---

# 1. Proč bylo řešení potřeba

Standardní SAP aplikace **Asset Master Worklist (F1592)** pracuje s daty z CDS view **`C_FixedAssetWorklist`**.  
Aplikace nemá zabudované žádné řízení přístupu podle lokace majetku (budovy).

Zákazník však požaduje:

- Uživatel smí vidět **jen majetek umístěný v objektu(budově)**, na který má oprávnění.  
- Každý uživatel může mít přístup na více objektů.  
- Hodnota objektu je uložena v tabulce **`ZAM_OBJEKT`**.  
- V datovém modelu Fiori aplikace se tato hodnota nachází v poli **`VehicleLicensePlateNumber`**.  
- Řádky **bez hodnoty** (prázdné KFZKZ) musí být viditelné všem.

Technicky jde o potřebu rozšířit **CDS Access Control**, aniž bychom zasahovali do standardních SAP autorizačních objektů.

---

# 2. Architektura řešení

Celé řešení stojí na třech prvcích:

1. **Vlastní autorizační pole** `ZKFZKZ`  
   – reprezentuje budovu/objekt, na který může být uživatel autorizován.

2. **Vlastní autorizační objekt** `Z_OBJ_KFZ`  
   – umožňuje přes PFCG definovat oprávnění pro jednotlivé objekty.

3. **Vlastní CDS DCL role** `ZR_FAA_KFZKZ`  
   – zapojuje autorizaci do CDS view `C_FixedAssetWorklist`  
     (rozšiřuje standardní kontroly SAP o námi definovaný filtr).

4. **Role v PFCG** (např. `ZAM_FAA_KFZKZ_10000001`)  
   – obsahuje povolené hodnoty KFZKZ a tím určuje, které objekty smí uživatel vidět.

Toto řešení:
- je **plně upgrade‑safe**,  
- **bez zásahu do standardních SAP objektů**,  
- **čistě konfigurační/data-driven**,  
- funguje automaticky v OData v2 i v2 UI5 aplikacích.

---

# 3. Tvorba autorizačního pole `ZKFZKZ` (SU20)

Autorizační pole musí existovat ještě předtím, než jej lze použít v autorizačním objektu. Pole je založeno na datovém elementu, který odpovídá skutečnému datovému typu KFZKZ – tedy identifikátoru budovy.

### Postup krok za krokem:

## 3.1 SE11 – využití domény `AM_KFZKZ`
Účel domény:
- definovat technické vlastnosti pole,
- zajistit F4 nápovědu přes Value Table.

**Kroky:**
1. `SE11` → Domain → Create  
2. Název: **`AM_KFZKZ`**
3. Typ: `CHAR`
4. Délka: dle délky objektu v `ZAM_OBJEKT` (15)
5. **Value Table:** `ZAM_OBJEKT`  
   → díky tomu získá pole F4 nápovědu v PFCG
6. Texty doplnit
7. Activate

---

## 3.2 SE11 – převzetí Data Elementu `AM_KFZKZ`
Účel DE:
- pojmenování pole,
- připojení search-help,
- použije se v SU20.

**Kroky:**
1. `SE11` → Data Element → Create  
2. Název: **`AM_KFZKZZ`**
3. Doména: `AM_KFZKZ`
4. Popisky: „Objekt“, „Kód objektu (KFZKZ)“ apod.
5. Activate

---

## 3.3 SU20 – vytvoření autorizačního pole `ZKFZKZ`
Účel:
- SAP použije toto pole v rámci SU21 objektu i v PFCG.

**Kroky:**
1. `SU20` → Create Authorization Field  
2. Název: **`ZKFZKZ`**
3. Data element: **`AM_KFZKZ`**
4. Organizational Level: NE
5. Uložit + Generate

---

# 4. Tvorba autorizačního objektu `Z_OBJ_KFZ` (SU21)

Tento objekt umožní definovat přístup na úrovni jednotlivých budov/objektů.

### Co objekt řeší:
- uživatel má přístup k jedné nebo více hodnotám KFZKZ,
- tyto hodnoty se spravují v PFCG,
- bude volán z CDS Access Control přes `pfcg_auth()`.

### Postup:

1. `SU21` → Create Authorization Object  
2. Název: **`Z_OBJ_KFZ`**
3. Authorization Fields:
   - `ACTVT` (standardní pole, hodnoty jako 03 = Display)
   - `ZKFZKZ` (námi vytvořené pole)
4. Dokumentace: popsat, že jde o řízení přístupu k majetku podle objektu.
5. Save & Generate

---

# 5. Tvorba PFCG role `ZAM_FAA_KFZKZ_10000001`

Tato role reprezentuje **konkrétní budovu/objekt** a obsahuje povolené hodnoty v našem novém autorizačním objektu.

### Účel role:
- řídit, které budovy může uživatel vidět v Asset Worklistu,
- oddělit oprávnění dle jednotlivých poboček/okresů,
- umožnit jednoduché přiřazení uživateli.

### Postup:

1. `PFCG` → Create Role  
2. Název: **`ZAM_FAA_KFZKZ_10000001`**
3. Režim *Authorizations*:
   - kliknout **Change Authorization Data**
   - najít objekt **`Z_OBJ_KFZ`**
   - nastavit:
     - `ACTVT` = **03**
     - `ZKFZKZ` = **10000001**  
       (hodnota objektu z tabulky `ZAM_OBJEKT`)  
4. Generate
5. Uložit
6. Přiřadit testovacímu uživateli

Tuto logiku lze zopakovat pro dalších ~100 objektů (pokud budou spravovány jako samostatné role).

---

# 6. Tvorba CDS Access Control (DCL) `ZR_FAA_KFZKZ`

Toto je nejdůležitější část řešení – **zavádí vlastní filtr do zobrazení dat**.

### Proč je potřeba:
- Standardní CDS Access Control u view `C_FixedAssetWorklist` kontroluje jen standardní SAP autorizační objekty.
- Potřebujeme dodat **další podmínku**, která zajišťuje filtr nad KFZKZ.
- Přitom standardní DCL **musí zůstat beze změny**.

Vlastní DCL role **se vyhodnocuje společně se standardem (AND)**, což je ideální chování.

### DCL soubor:

```abap
@EndUserText.label: 'Extra access: KFZKZ filter for C_FixedAssetWorklist'
@MappingRole: true
define role ZR_FAA_KFZKZ {

  grant select on C_FixedAssetWorklist
    where
      // Prázdné hodnoty jsou viditelné všem uživatelům
      VehicleLicensePlateNumber is null
      or VehicleLicensePlateNumber = ''

      // Jinak se vyhodnocuje oprávnění podle PFCG role
      or ( VehicleLicensePlateNumber ) =
           aspect pfcg_auth( Z_OBJ_KFZ, ZKFZKZ, ACTVT = '03' );
}
```

### Co přesně dělá:

1. Pokud je `VehicleLicensePlateNumber` prázdné → záznam projde.  
2. Pokud má hodnotu → DCL ověří, zda má uživatel v rolích PFCG povolenu tuto hodnotu v `Z_OBJ_KFZ`.  
3. Pokud ano → záznam projde.  
4. Pokud ne → záznam se **odfiltruje**.

---

# 7. Přístupová logika – shrnutí pro business

- Každý majetek má přiřazenou **budovu/objekt** (KFZKZ).  
- Každý uživatel má přiřazené **jedno nebo více oprávnění** na konkrétní objekty.  
- Ve Fiori aplikaci uvidí uživatel **pouze majetek**, který se nachází v objektech, na které má oprávnění.  
- Majetek bez přiřazené budovy (KFZKZ prázdné) je viditelný všem.  
- Logika funguje automaticky ve všech prostředích.

---

# 8. Kompletní end-to-end tok

1. Majetek v SAP má hodnotu KFZKZ → ta se propsala do `VehicleLicensePlateNumber`.  
2. Uživatel má v PFCG přiřazenou roli (např. `ZAM_FAA_KFZKZ_10000001`).  
3. Tato role mu povoluje vidět objekt **10000001**.  
4. Otevře Fiori aplikaci → UI5 volá CDS/OData.  
5. CDS Access Control:
   - vyhodnotí standardní SAP role,  
   - vyhodnotí naši Z‑DCL,  
   - povolí pouze záznamy s KFZKZ = 10000001 nebo prázdné.  
6. Uživatel uvidí jen relevantní data.

---

# 9. Výhody řešení

- Bez zásahu do standardního kódu SAP.  
- Pouze konfigurace (SU20, SU21, PFCG) + CDS DCL.  
- Jasné řízení přístupu na úrovni „lokací majetku“.  
- Možnost udržet až stovky kombinací přes jednoduché role.  
- Přenositelné mezi systémy (transport).  
- Škálovatelné pro další business jednotky.

---

# 10. Doporučení do budoucna

- Hodnoty v `ZAM_OBJEKT` udržovat konzistentně (bez duplicit).  
- Pro větší objemy (např. 300+ objektů) zvážit semi‑automatizovanou tvorbu rolí.  
- Pokud by se někdy řešilo „k datu výkazu“, lze rozšířit o časově platnou tabulku (časová varianta DCL).

---

# 11. Závěr

Celé řešení umožňuje přesné řízení přístupu k majetku na úrovni budov/objektů, je implementačně nenáročné, čisté, bezpečné a dobře udržovatelné.  
Pro business znamená zásadní snížení rizika nežádoucího přístupu a správné rozdělení zodpovědností.

